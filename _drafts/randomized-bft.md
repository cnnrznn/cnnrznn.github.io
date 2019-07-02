---
layout: single
title:  "Randomized BFT Consensus"
header:
tags:
  - distributed systems
---

In an early post, I tackled the problem of byzantine fault tolerant (BFT)
broadcast by implementing an algorithm by Bracha. In this post, I address the
problem of BFT consensus by explaining and implementing an algorithm from the
same paper.

Here, I address consensus. A group of $n$ processes want to agree on the value
of a variable. Here, I present an algorithm for doing so which is optimally
resilient to $f$ byzantine processes where $n >= 3f + 1$.

## The System Model

### Failures

In Bracha's paper, they call what is now known as _byzantine_ behavior as
explicitly _malicious_. There is a subtle but very important distinction between
the two. Although a failure could be malicious, it might also be caused by bad
programming, bit flips, or other noise in the system. Failed processes in this
system model can behave arbitrarily, changing their state or sending arbitrary
messages.

### Network

For the algorithm to work, processes _must_ be able to determine the sender of
every message. Otherwise, a malicious process could impersonate the rest of the
system. In the implementation, this is done "honestly", i.e. sender addresses
are taken as ground truth. In practice, this could be done with crypto
signatures.

## The Algorithm

The algorithm begins with each process proposing a value and proceeds through a
series of rounds. In each round, each replica learns more about what everyone
else knows, using thresholds to filter out malicious actors.
