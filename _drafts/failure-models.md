---
layout: single
title:  "Models of Failure in Distributed Systems"
header:
  teaser: "unsplash-gallery-image-2-th.jpg"
tags:
  - distributed systems
---

In distributed systems, protocols and algorithms are designed with regards to a particular set of assumptions.
One of these assumptions is the *failure model* of components of the system.
For example, when designing a protocol we might have assumptions how processes fail, and a different assumption about the types of network failures.
In this article, I present a survey of the different failure models used in distributed systems.

## The Consensus Problem in Unreliable Distributed Systems
* crash faults
* byzantine faults
* network connectivity
* network reliability
* synchrony
* signatures
