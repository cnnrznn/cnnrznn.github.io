---
layout: single
title:  "The Raft Consensus Protocol"
header:
  teaser: "unsplash-gallery-image-2-th.jpg"
categories: 
  - distributed systems
  - golang
tags:
---

Raft is a protocol that solves the distributed systems problem of consensus.
Specifically, Raft allows a network of machines to agree on the entries in an
append-only log. When an entry is committed to the log on one machine, the
protocol gaurantees no other machine will ever commit a different entry at the
same index.

## Assumptions

As with all distributed system protocols, Raft makes assumptions about its
adversaries and failed nodes.  Raft is a _crash fault tolerant_ (CFT) protocol,
which means that failed nodes can only fail by suddenly terminating execution.
For example, another failure mode is _byzantine fault tolerance_ (BFT). In BFT,
nodes can fail by executing arbitrary operations and can even be _malicious_.
Raft, being a CFT protocol, assumes that every node is benign and therefore
every node that is still "awake" follows the protocol exactly.

When designing distributed systems, we define the number of nodes needed for
correct operation of the protocol in terms of ***f***, the number of expected
failed nodes.  Under CFT, a system needs a minimum of `2f + 1` nodes to operate
correctly. What ***f*** is is up to the system maintainers. It is important to
note that if  the number ***f*** is exceeded, assumptions about the correctness
of the protocol no longer hold. For example, if we are maintaining a cluster of
5 nodes, the protocol will work as long as no more than 2 nodes fail.

## Consensus

Consensus is the process in distributed system of achieving agreement on a fact
or set of facts. In the context of Raft, this agreement is on the entries in a
log, an append-only list.

Typically, consensus protocols have two sub-protocols: leader election and
"normal operation."

### Leader Election
Having a leader allows for easier serialization of log entries, but requires a
protocol for electing a new leader on leader failure.

The leader election protocol in raft can be summarized as:

> Which node does a simple majority believe has the most up-to-date log?

When a node initiates an election, it tells the other machines its (1) highest
index and (2) the term that entered that index. Based on this, peer machines
will vote for the candidate if the term is greater than its own, or the terms
are equal and the other machine's log is at least as long.

### Log Append
Normal operation is a bit of a misnomer (the whole protocol is normal
operation), but refers to the part of the protocol responsible for accepting
input from the user and coming to consensus on the input.

In Raft, log append can be summarized as:
> For each follower, the leader sends the follower all un-replicated log entries.
> The follower appends the log entries and responds.
> The leader sets the commit index to ***n*** when a simple majority of the followers have successfully replicated ***n***.
> The follower sets the commit index to ***n*** when they learn the leader has a committed index of ***n***.

Because there are `2f + 1` nodes in the cluster, if a message is replicated by a
simple majority, `f + 1`, the committed entry is gauranteed to be able to
survive if ***f*** nodes fail. If ***f*** nodes that replicated the log entry
fail but one survives, it will be able to claim leadership because it has the
most up-to-date log. Subsequently, as leader it will be able to replicate that
log entry to the surviving followers.

# Distributed systems and Go

I've implemented the Raft protocol using Go
[here](https://github.com/cnnrznn/raft).  Go's support for concurrency makes it
a natural fit for distributed system development.  Let's take a look at how it
was done.

## Repository organization

![a preview](/images/raft-consensus/repo.png)

- `cmd/httpraft` An executable that provides a HTTP API
- `cnet` Networking library for the raft protocol
- `client.go` Go API for raft
- `messages.go` Definitions of protocol messages
- `raft.go` Core protocol
- `util.go` Random useful functions

## Architecture

![Go Architecture](/images/raft-consensus/go-arch.png)

In the above diagram, each circle represents a goroutine.  We can see that the
Raft module is designed to be run in a separate goroutine from the client
program.  The API functions in `client.go` will be run in the same goroutine as
the client, and handle communication with the protocol through unexposed
input/output channels.

The core Raft logic receives input from the client code and the `cnet`
networking module.  It executes logic based on received messages and timeouts,
and sends output to `cnet`.  As with the client library and Raft, the Raft logic
runs the networking module in a separate goroutine and communicates messages to
send and those received by channels.

Finally, the networking library runs two goroutines. The main goroutine executes
the send and receive logic while the second listens for incoming connections. When the
listening goroutine accepts a connection from a client, the connection is passed to
the main `cnet` goroutine so it can execute the receive logic.

## `httpraft`

`httpraft` is a binary written in Go that exposes the operations of the log
through a HTTP API. I have written [a small Postman
collection](https://github.com/cnnrznn/raft-postman) that shows how `httpraft`
may be used by an external, (not-necessarily-golang) client.

# Usage

1. Download [this repo](https://github.com/cnnrznn/raft) and build `cmd/httpraft`
2. Download [the postman repo](https://github.com/cnnrznn/raft-postman)
3. Create a json configuration for the network, like so:
```json
{
    "peers": [
        "localhost:8000",
        "localhost:8001",
        "localhost:8002"
    ],
    "apis": [
        "localhost:9000",
        "localhost:9001",
        "localhost:9002"
    ]
}
```
Call this file `peers.json` and put it in the working directory where you will run `httpraft`.
4. Run the binary with `./httpraft <id>` where `id` is the instance of the protocol.
5. Point the postman collection to any of the endpoints in `apis`.
