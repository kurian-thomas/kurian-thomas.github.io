+++
title = "A Brief Outline Of A Mini Transaction Processing Unit Backed By RocksDB In Go"
date = 2026-07-25T21:03:00-07:00
draft = false
group = "golang/transaction-processing-unit"
+++

<div class="callout-note">
<p><strong><em>*** Note ***</em>:</strong></p>
  <p><em>This is a very high-level brief, as going over every single design choice would be madness (also I don't remember most of it). This is also not a guide nor does it have best practices, there are questionable design choices, which I am happy to discuss. Finally, the goal was to compare these two concurrency control mechanisms, and the setup was such a cool thing to build that I had to write about it.</em></p>
</div>

### Motivation

<i>It was part of a final project for a course in my master's program. Learned a whole lot, and it really changed my impression of using Go. One of the most productive languages ever.<br>
</i>

### Let's Dive In

#### Goal

<i>
The course was about distributed transaction processing, and our goal was to compare concurrency control mechanisms to understand and evaluate the trade-offs.<br>
<br>
My teammate and I presented it as a conference-style paper titled Comparison Of Optimistic Concurrency Control (OCC) And Conservative 2-Phase Locking (C2PL) Using A Multi-Threaded Transaction Processing Unit Over RocksDB.
</i>

### High Level Architecture

![high-level-arch-tpu](assets/posts/golang/transaction-processing/high-level-arch-tpu.webp)

<i>
The system had 2 major pieces:<br>

1. Client: A lightweight compiler that converted a DSL (domain-specific language) text file into an intermediate language, which was a mix of expression logic and reserved keywords that the server understood, then sent via REST. The fun part was actually learning how to use context-free grammar for parsing this DSL.<br>We will talk about this further down.

2. Server: Typical Golang HTTP server. This was a transaction processing unit layer that sat in front of the RocksDB storage layer. It basically routed requests to either OCC or 2PL (strict) concurrency control paths and controlled worker resources (threads) that connected to RocksDB. This is where Golang shined. It was such a pleasure to code and conceptualize.
   </i>

#### The Client

![client-arch](assets/posts/golang/transaction-processing/transaction_processing_unit_client_arch.webp)

![skeletor-laugh](assets/posts/memes/skeletor-laugh.gif)

<i>
What is this monster? Well, this is how we decided to parse the workload DSL. Built a tiny compiler!! with context-free grammar!! and then used that template to hydrate it with a key set so that the client could generate multiple requests with different random keys from a set and an overlapping hot set. i.e., if the client selects a key from the hotset (high contention) with probability p, then it selects a key from the normal set with probability 1-p. This is particularly interesting as contention is a huge factor in deciding which CC we need to pick.<br>

So what's happening here?<br>In a nutshell, it converts a DSL -> intermediate language -> hydrates it with random keys picked from a set and a hotset with a certain probability.<br>

eg: DSL <br>
</i>
Insert to Database:

```text
INSERT
KEY: <key>, VALUE: <value: map>
...
END
```

Transaction:

```text
TRANSACTION (INPUTS: <key1>, <key2>, ...)
BEGIN
value1 = READ(key1)
<logic>   // e.g., value2 = value1 + 1
WRITE(key2, value2)
...
COMMIT
```

Workload:

```text
WORKLOAD
<TRANSACTION 1>
<TRANSACTION 2>
...
END
```

Intermediate Parsed Template (Human Readable):

![intermediate-parsed-template](assets/posts/golang/transaction-processing/transaction_processing_unit_client_parsed_template.webp)

Intermediate Hydrated Request To Server (Human Readable):

![intermediate-parsed-hydrated-template](assets/posts/golang/transaction-processing/transaction_processing_unit_client_parsed_hydrated_template.webp)

<div class="callout-note">
<i>
What does the intermediate form mean?<br>
1. Get -> Get from the database<br>
2. Code -> expression logic that is compiled/evaluated on the server for arithmetic/business updates<br>
3. Set -> Set a value to a key in the database<br>
</i>
</div>

<i>
This is then packaged into a format the server understands and sent via REST. In hindsight, I should have used an RPC protocol, it would have been faster and more compact, but I was hacking through and kept it simple.
</i>

#### The Server

![intermediate-parsed-hydrated-template](assets/posts/golang/transaction-processing/transaction_processing_unit_server_multi_threaded.webp)
![ron-beautiful](assets/posts/memes/ron-its-beautiful.gif)

<i>
Ain't she beautiful?<br>
<br>
I designed it based on 2 core concepts: the actor queuing strategy and the channel-based fan-out strategy.<br>
<br>
Why?<br>
1. I needed each query to `wait` for the TPU to translate and execute the query or die (abort) and retry (we will talk about this later)<br>
2. I needed to control the number of goroutines (Go runtime-managed lightweight threads multiplexed over OS threads), because cgo calls into RocksDB can pin/work with OS threads. Controlling concurrency was needed for metrics, and also to see how the system improved as I increased thread count, while assuming limited server resources.
</i>

#### Go Multithreading Model Brief

<i>
Go uses an M:N model where M goroutines are mapped to N OS threads. One of the greatest things in goroutines is channels, as they provide an OOTB wait behavior when full/empty, which suspends the goroutine. This way we can make sure the server does not run out of CPU resources. Productivity and conceptual understanding at its peak.
</i>
<br>
<br>

<i>
So what does it do?<br>
In a nutshell: <br>
1. Takes the request and scans it to identify the keys beforehand.<br>
2. It generates a common context data struct that is used to pass information around.<br>
3. Tries to acquire locks before doing anything (2PL path). If it fails, retry with timeout & backoff.<br>
4. Translate the query and execute it.<br>
</i>

### The Cool Parts (Server Internals)

![intermediate-parsed-hydrated-template](assets/posts/golang/transaction-processing/transaction_processing_unit_processing_unit_server_internal.webp)
![nerd-excited](assets/posts/memes/nerd-excited.gif)

<i>
The HTTP server has 2 main workload routes (InitDb, and Runschedule):<br>

1. The InitDb route -> Inserts all the possible keys. _Hack_ so that we don't have to handle missing-key errors everywhere.<br>
2. The execute workload route -> The request tells the server, "Hey, execute this query in 2PL or OCC (SI-style validation)."
   </i>

### Let's Break Down What 2PL & Snapshot Isolation Mean Here

### 2 Phase Locking

<i>
We are using a conservative 2PL style with strict release, which means we know the keys beforehand and acquire all locks before doing anything.<br>
<br>
2PL, as the name suggests, has 2 phases.<br>
1. The growth phase -> acquiring all the locks<br>
2. The shrink phase -> releasing all the locks<br>
<br>
There is no in-between: if it gets all the locks, then success!! It can do all the logic on the values and change them, then release everything. What's great is that it guarantees protection, i.e., while this transaction is changing keys, no other transaction can change them until this is done (because they cannot acquire locks on those keys).
</i>

### Snapshot Isolation (OCC)

<i>
I don't think this is exactly textbook Snapshot Isolation, but it's close.<br>
<br>
The idea is simple: we don't need strict lock-holding during execution, we use glorious versioning!!!!! That means if transaction T1 reads a key with version V0, does its logic, and then wants to write, it validates that the relevant versions are still compatible before committing. In our implementation, validation uses read/write sets plus version checks, and commit applies writes atomically with a commit timestamp. It asks the question: did any other transaction write to this key while I was doing my logic, making my read stale? We have mathematical set-based guarantees like read-set/write-set validation, which I won't dive into here.
</i>

### How Do We Avoid Deadlocks While Operating In 2PL

<i>
Deadlocks can occur when there are circular waits, i.e., T1 waits for locks on key K1 while holding K2, and T2 waits for locks on K2 while holding K1. There is deadlock detection, but prevention is a super cool algorithm. We chose to use the Wait-Die algorithm.<br>
</i>
<br>
<i>
What does it do?<br>
We provide an incremental timestamp/ID for every transaction. Now we can categorize them as newer or older by comparing timestamp/ID.<br>
<br>
The Wait-Die algorithm:<br>
1. Ti has higher priority than Tj when Ti is older than Tj (Tj has the greater timestamp).<br>
If Ti wants a lock held by Tj, Ti waits (old waits for young).<br>
If Tj wants a lock held by Ti, Tj aborts and retries with timeout & backoff. literally Dies!!!

Better explanation is here: [Lecture 19 Part 6 Deadlock Avoidance](https://www.youtube.com/watch?v=xbMqNpW5VKQ)<br>
Illustration from the lecture:
</i>

![wait-die-illustration](assets/posts/golang/transaction-processing/wait-die.png)

### How Did The Server Execute The Intermediate Code

<i>
This was also a pretty hacky way of implementing this.<br><br>
Every GET and SET is a DBTask.<br>
So, the server iterated through parsed GET operations and used the operation key to fetch from RocksDB. The fetched result was stored in an in-memory cache (a simple
map per translation, no complicated eviction policy included).<br>
Every CODE block was split into LHS and RHS, where the RHS was compiled and evaluated against the variable values in the cache and then stored back. (The rule here is that every variable should be fetched via GET before doing anything with it.)<br>
Finally, every SET fetches the final value from the cache and updates the RocksDB entry.<br><br>
We also built a wrapper around RocksDB to make it versioned, so every entry gets a version assigned to it.
</i>

### Interesting Optimization

<i>
I noticed large GC spikes when load testing the server. My guess was frequent `DBTask` allocation churn causing this. An awesome way Golang helps with this is object pooling (`sync.Pool`) to reuse task objects across requests. So the server reuses objects for this use case, and GC spikes are reduced.
</i>

### Load Testing and Observability

<i>
So we had to load test this and derive metrics to compare these 2 concurrency control mechanisms. I am not going over the hacky metrics observability I used (hint: write metrics to a few global CSV files xP). I will go over the test runner and a few observations.<br>
</i>

![python-test-runner](assets/posts/golang/transaction-processing/transaction_processing_unit_test_runner.webp)

![python-test-runner-ss](assets/posts/golang/transaction-processing/test_runner_ss.png)

<i>
Since we could compile the client and server as individual binaries, we could also run them with separate CLI configurations. We ran a whole bunch of tests sweeping through different configurations, e.g., number of threads and contention levels with varying probabilities. Mainly used Bash scripts for exercising the binaries and a Python test runner as an orchestrator, which ran the whole suite, gathered metric files, and used matplotlib to visualize it.
</i>

#### Some Fancy Result Graphs

![2pl_commit_retry_count_v_response_time](assets/posts/golang/transaction-processing/2pl_log_distribution.png)
![occsi_commit_retry_count_v_reponse_time](assets/posts/golang/transaction-processing/occsi_log_distribution.png)

<i>
Multiple other tests were conducted in comparing these mechanisms. Seeing how each workload varied, and how architecture and system resources affected outcomes with trade-offs, was wonderful to observe. I do acknowledge this might not be the most exact or ideal setup to measure and compare. All in all, it was such a great experience building this and learning a lot in the process. Truly appreciate it.

![cloudy-cry](assets/posts/memes/cloudy-cry.gif)
</i>

### Remarks

<i>
As I am typing this up, I am realizing how awesome and hard this project was. Learned a lot!! I am losing steam writing about it, but I think it's a really cool first blog article. I have been sitting on it for a while now, and finally thought of writing about it. Let me know if you found this nerdy and cool. It was a lot of fun and really made me fall in love with Go.
</i>

### Acknowledgments (The MVPs)

[Professor Faisal S. Nawab](https://www.linkedin.com/in/faisal-nawab-38597333/), and TAs [Farzad Habibi](https://www.linkedin.com/in/habibif/) and [Yinan Zhou](https://www.linkedin.com/in/yinandanielzhou/), for instruction and guidance at the University of California, Irvine, and my teammate [Henry Keane](https://www.linkedin.com/in/hkeane/) for his support in making this project a success.
