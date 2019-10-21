---
layout: single
title:  "Developing Distributed Systems in Golang"
header:
  teaser: ""
categories: 
  - distributed systems
published: false
---

Learning to program distributed systems can be difficult. If one is starting
from ground zero, there are two problems to tackle simultaneously. The first is
***message retransmission***. Faulty networks mean that one has to worry about
unreliable networks between processes. At the same time, one to implement
correctly the ***protocol logic***. To make developing distributed systems
easier, I present `dsdriver`, a library for developing and deploying distributed
systems written in Golang.

[github.com/cnnrznn/dsdriver](https://github.com/cnnrznn/dsdriver)

## The Inspiration
The convention in papers from the 80's that give distributed system protocols is
to abstract the network as a buffer of messages with the following semantics:

- $send(p, m)$
  - Send message $m$ to process $p$
- $recv(p)$ :
  - Receive a message $\\{m, \phi\\}$ for process $p$
  - May return the null message, $\phi$, a finite number of times (delay)

This convention is used to abstract unreliable networks from the protocol logic.
Assuming that messages are *send once-receive eventually* is possible because of
retransmissions. We'll address this later.

My inspiration for this work is to be able to implement seminal works **(1) as
they are described** in their respective papers. In addition, I want to be able
to test for **(2) delay and reordering** of messages. Finally I want to be able
to **(3) monitor and evaluate** works for bandwidth usage.

## Using the `dsdriver`
### Installation
`dsdriver` can be installed through the `go` tool:

```bash
go get github.com/cnnrznn/dsdriver
```

### Usage

