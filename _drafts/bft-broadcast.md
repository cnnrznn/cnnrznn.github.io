---
layout: single
title:  "Byzantine Fault Tolerant Broadcast"
tags:
  - distributed systems
  - coding
---

An essential primitive in distributed systems is _broadcast_, or the ability to send a group of participants the same message.
In the honest case, this problem is straightforward; send everyone the message and acknowledge its receipt.
However, this problem becomes complicated under the assumption of byzantine faults.
If we apply the honest protocol to the byzantine environment, we may have different peers accept different values!
In this article, I give a summary and implementation of an algorithm for overcoming the problem.

## The Problem
In this instance we consider the _byzantine fault model_.
That is, participants in our distributed system can behave arbitrarily.
Our only gaurantee is that _honest_ participants - peers from here on - obey the protocol.

The other type of failure model usually considered is that of the network.
That is out of scope for this article.
We will simply assume here that messages from honest peers are delivered eventually.
Later, I will show how the implementation copes with messages from dishonest (byzantine) peers.
Then we will see that exactly-once delivery of messages from honest peers is nice, but not required.

## The Code
If you would like to follow along during this post, the code can be found at https://github.com/cnnrznn/clj-net.git.
The code for this post is located in the `src/clj_net/broadcast.clj` file.
The project is written in Clojure which is a beautiful functional language that is also practical as it runs on the JVM and has access to the Java ecosystem.
If you wish to run the code, you'll need [Leiningen](https://leiningen.org/).

## The Solution
The solution to the broadcast problem is to insert another layer in the stack: a reliable broadcast primitive.
The goal of the primitive is to reduce the power of byzantine processes to that of fail-stop processes.
Two properties must hold for a process _p_ broadcasting message *v*:

1. If *p* is correct, all correct processes agree on *v*
2. If *p* is faulty, then either all correct processes agree on the same value, or no value is accepted from *p*

This protocol comes from an old paper by Gabriel Bracha _[Asynchronous Byzantine Agreement Protocols](https://core.ac.uk/download/pdf/82523202.pdf)_.
After establishing this primitive, bracha uses it to achieve a randomized consensus algorithm resilient to *n >= 3f+1* faults!
