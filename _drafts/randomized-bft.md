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

Here, I address consensus. A group of $n$ processes each start with a variable
and want to agree on the value of that variable. Here, I present an algorithm
for doing so which is optimally resilient to $f$ byzantine processes where $n >=
3f + 1$.
