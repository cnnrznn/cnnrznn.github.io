---
layout: single
title:  "A Distributed System Developing Framework for the Masses"
header:
  teaser: ""
categories: 
  - distributed systems
---

Learning to program distributed systems can be difficult. If you are starting
from ground zero, there are two problems to tackle simultaneously. The first is
message retransmission. Faulty networks mean that the developer has to worry
about doing non blocking message transmission between processes. At the same
time, they must handle the protocol logic. To make developing distributed
systems easier, I present `dsdriver`, a library for developing and deploying
distributed systems written in Golang.

## The inspiration
- assume a send will eventually be received
- be able to test re-order, delay capabilities
