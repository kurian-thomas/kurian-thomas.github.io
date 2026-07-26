+++
title = "Mini Transaction Processing Unit Backed By Rocks DB In Go"
date = 2026-07-25T21:03:00-07:00
draft = true
group = "golang/transaction-processing-unit"
+++

### Motivation

<i>It was part of a final project for a course for my masters. Learnt a whole lot and really changed my impression on using Go. One of the most productive languages ever.</i>

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

1. Client: Which is a light-weight compiler which converted a DSL text file to an intermediate language which was mix of Golang code and reserved key-words that the server understood and sent via rest. The fun part was actually learning how to use context free grammar for parsing this DSL,<br>We will talk about this further down.

2. Server: Typical Golang http server. This was a transaction processing unit which is the layer that stood before the RocksDB storage layer. It basically routed the request to use either OCC or 2PL (strict) concurrency control paths and controlled resources (threads) that connected to RocksDB. This is where Golang shined. It was such a pleasure coding it up and to conceptualize.
   </i>

#### The Client
