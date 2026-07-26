+++
title = "A Brief Outline of a Mini Transaction Processing Unit Backed By Rocks DB In Go"
date = 2026-07-25T21:03:00-07:00
draft = true
group = "golang/transaction-processing-unit"
+++

<div class="callout-note">
<p><strong><em>*** Note ***</em>:</strong></p>
  <p><em>This is a very high level brief, as going over every single design choice would be madness (also I don't remember most of it). This is also not a guide or best practices,there would be questionable design choices for which I am happy to discuss. Finally the goal was to compare these two concurrency control mechanisms and the setup was such a cool thing to build that I had to write about it.</em></p>
</div>

### Motivation

<i>It was part of a final project for a course for my masters. Learnt a whole lot and really changed my impression on using Go. One of the most productive languages ever.<br>
</i>

### Lets Dive In

#### Goal

<i>
The course was dealing with distributed transaction processing and out goal wad to compare concurrency control mechanisms to understand and evaluate the trade-offs.<br>
<br>
Me and my teammate present it like a conference style paper titled Comparison Of Optimistic Concurrency Control (OCC) And Conservative 2 Phase Locking (C2PL) Using A Multi-Threaded Transaction Processing Over RocksDB.
</i>

#### High Level Architecture

![high-level-arch-tpu](assets/posts/golang/transaction-processing/high-level-arch-tpu.webp)

<i>
The system had 2 major pieces:<br>

1. Client: Which is a light-weight compiler which converted a DSL (Domain Specific language) text file to an intermediate language which was mix of Golang code and reserved key-words that the server understood and sent via rest. The fun part was actually learning how to use context free grammar for parsing this DSL,<br>We will talk about this further down.

2. Server: Typical Golang http server. This was a transaction processing unit which is the layer that stood before the RocksDB storage layer. It basically routed the request to use either OCC or 2PL (strict) concurrency control paths and controlled resources (threads) that connected to RocksDB. This is where Golang shined. It was such a pleasure coding it up and to conceptualize.
   </i>

#### The Client

![client-arch](assets/posts/golang/transaction-processing/transaction_processing_unit_client_arch.webp)

<i>
What is this monster ? Well this is how we decided to parse the workdload DSL. Built a tiny compiler !! with context free grammar !! and then used that template to hyderate it with key a key set so that the client can generate multiple requests with different random keys from a set and an overlaping hot set. ie. If the client selectes a key from the hotset (High contention) with probability p then it will select a key from the normal set with probability 1-p. This is particularly interesting as contention is a huge factor on which CC we need to pick. <br>

So Whats happening here.<br> In a nutshell it is converting a DSL -> intermediate language -> hyderate it with random keys picked from a set and a hotset with a certain probability. <br>

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

Intermediate parsed template (human readable):

![intermediate-parsed-template](assets/posts/golang/transaction-processing/transaction_processing_unit_client_parsed_template.webp)

Intermediate hydrated request to server (human readable):

![intermediate-parsed-hydrated-template](assets/posts/golang/transaction-processing/transaction_processing_unit_client_parsed_hydrated_template.webp)
