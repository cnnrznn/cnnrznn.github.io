---
layout: single
title:  "Models of Failure in Distributed Systems"
header:
  teaser: "unsplash-gallery-image-2-th.jpg"
tags:
  - distributed systems
---

In distributed systems, protocols and algorithms are each designed with regards to a particular set of assumptions.
One of these assumptions is the *failure model* of components of the system.
For example, we might make assumptions about how processes fail, and others about how the message-passing system, the network, fails.
These assumptions are critical as they provide us with knowledge of the *capabilities* of protocols as well as a means of comparing them.
In this article, I present a survey of the different failure models used in distributed systems.

## The Consensus Problem in Unreliable Distributed Systems
* crash faults
* byzantine faults
* network connectivity
* network reliability
* synchrony
* signatures
* scheduling

### A Note on Lingo
In distributed systems, often we reduce the system mentally to a collection of *nodes* which represent processes and *edges* which represent the network link(s) between processes.
In the real world, these entities are more complicated than lines and dots.
However, we can think of them simply as such and charactize the complexity in terms of a failure model.
Neat-O!

Throughout this piece I may refer to a ***run***.
A run is simply an execution of a protocol, from start to finish.
While we could also make assumptions about how components behave outside of a run, here I am restricting the scope to the failure model during a run.

## Processing Faults
A typical distributed protocol is implemented as follows.

```
state = init()
loop:
  broadcast(state)
  messages = receive()
  state = tick(state, messages)
```

Processing faults (process faults, node faults) occur at the nodes of the system, where the *execution* of the above code is performed.
The failure model in which no faults occur is sometimes reffered to as the **benign** failure model.
Put another way, processes can be said to be *correct*, *honest*, or *benign* if they operate according to the benign failure model.
This is the strongest assumption, and also the least attainable in practice.
Under the benign failure model, all processes operate exactly according to the protocol for the duration of the run.

The next kind of processing failure model is the **crash** failure model.
The crash model is weaker than the benign model.
Under the crash model, processes execute according to the protocol.
At some point during the run, an evil gremlin comes along and sends `SIGKILL` to the processes.
For the rest of the run, the process does no work and does not send any messages.
In other words, processes can die, but before they die they are perfect little angels that can do no wrong (only the good die young).

The final kind of processing failure model is the **byzantine** failure model.
This model is the weakest of the three, but most closely echoes reality.
Under the byzantine failure model, processes are allowed to execute arbitrarily.
Processes can decide to follow the protocol, if they want to, but maybe they won't if they're having a bad day.
Processes can be ***honest***, or they can lie, cheat, collude, time travel, etc.
In other words, no one can be trusted, and most likely processes are stealing your money for Facebook.
