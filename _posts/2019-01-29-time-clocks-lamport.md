---
layout: single
title:  "Time, Clocks, and Leslie Lamport"
categories: 
  - distributed systems
---

In the paper _Time, Clocks, and the Ordering of Events in a Distributed System_, Leslie Lamport defines the _happens before_ relationship for distributed systems.
This is a seminal work that is fundamental to the understanding of distributed systems.
It addresses the simple yet profound problem of answering the question, "What happened when?"
This is an overview of that paper and a summary of the key ideas.

## System Model

A **distributed system** is comprised of a group of processes that communicate by sending messages.
Although the idea of a distributed system can be more general than that, for the scope of the paper the "nodes" are processes and the "edges" are messages.
Other examples of distributed systems include cells in the body, people and the mail service, a multiprocessor computer system, etc.
The term is a large umbrella.

An **event** in the system is the transmission or receipt of a message.
A **process**, therefore, can be specified by the ordered collection of events that occur during its lifetime.

## Partial Order
This paper introduces the _happens before_ relationship, one that can be used to define a partial order over a set of events.
For every pair of events, it can be said that either (1) one events happens before the other, or (2) the events happen concurrently.
In the latter case, it is impossible to tell which happened first because physical clocks cannot be trusted.

The happens before relationship for two events _a_ and _b_ is represented as a right-facing arrow '->' and is defined by three characteristics as follows:

1. If event _a_ comes before _b_ in the same process, _a -> b_.
2. If event _a_ is the sending of a message and _b_ is the receiving of the same message, then _a -> b_.
3. If _a -> b_ and _b -> c_, then _a -> c_.

These three rules give us a partial ordering over the set of events.
The proof of this is left to the paper and as an exercise for the reader :)

## Logical Clocks
Now that we have a partial order over events, we can assign a logical time to each event.
It is natural to write programs that rely on a clock, but in place of a physical time we can simply define some logical clocks based on our relationship.

Specifically, the paper defines a function _C(e)_ which gives the logical time for event _e_ in the distributed system.
There is but one requirement for this function, that it satisfies the _clock condition_ :

`if a->b then C(a) < C(b)`

This clock function can be defined by two rules:

1. A process increments its clock before sending a message.
2. A process receives a message, and then sets its clock to the maximum of:
  - its current clock plus one
  - the timestamp of the received message plus one

With these two rules, our clock function satisfies the clock condition.
Again, left as an excercise to the reader to prove this.

## Total Order

Now that we have a partial ordering in terms of clock values (you can create scenarios where two events have the same value), we can define an arbitrary, deterministic _total ordering over all events.
A trivial yet powerful method is to assign every process an identifier.
When two events have the same clock value, they must have been executed on different processes.
Therefore, we can order events by sorting again these events by their PID.

The paper does not define the ordering among processes, only that such an ordering exists; PIDs are a simple way to accomplish this.

The paper wraps up this section by using the total ordering property to solve distributed mutual exclusion.
I don't want to discuss it here because it reads more like a motivation for distributed total ordering than a core contribution of the paper.

## Physical Clocks

The method of total ordering can be further extended to synchronize distributed processes to some phyisical time, within some bound.

# Conclusion
