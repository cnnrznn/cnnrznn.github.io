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
***message retransmission***. Faulty networks mean that one has to worry about
message transmission between processes. At the same time, one has to implement
correctly the ***protocol logic***. To make developing distributed systems
easier, I present `dsdriver`, a library for developing and deploying distributed
systems written in Golang.

[The DSDRIVER source code](https://github.com/cnnrznn/dsdriver)

## The inspiration
- Abstracting unreliable network
  - assume a send will eventually be received
- Fault tolerance testing
  - be able to test re-order, delay capabilities
  - simulate latency
  - simulate delay (temporary partition)
- Monitoring
  - measuring number of measages, bandwidth usage

## 

## Using the `dsdriver`
### Installation
`dsdriver` can be installed through the `go` tool:

```bash
go get github.com/cnnrznn/dsdriver
```

### Usage

