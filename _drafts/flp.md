---
layout: single
title:  "FLP[DDS]: The Halting Problem of Distributed Systems
categories: 
  - distributed systems
---

In distributed systems, there is a fundamental theoretical problem which makes implementing distributed systems hard under the assumptions of the internet.
In a 1982 paper from MIT's Fischer, Lynch, and Paterson, it is proved the impossibilty of consensus in a distributed system in the presence of merely a single faulty process.
This problem is basically the halting problem of distributed systems.
In this post, I will give a summary of the proof and explain its impact on practical systems.

## The Consensus Problem
A fundamental problem in distributed systems, what I believe to be _the core problem_ in distributed systems, is consensus.
Suppose we have a system of processes that communicate by sending messages through a message channel.
The question is, can the correctly performing components of the system reach agreement?
If the answer to this question is no we are in trouble.
In this case, the best we could hope to do is to have every component guess what it thinks the other components know.
But this is dangerous; what if one computer thinks I sent $10 to Alice and another thinks I sent that money to Bob?
Oh no!

## Properties of Solution
A solution to the consensus problem must have three properties: _agreement_, _validity_, and _termination_:

* **Agreement** All pairs of correct processes decide the same value.
* **Validity** If all correct processes start with value _v_, then _v_ is decided.
* **Termination** All correct processes decide on a value.

## Failure Modes
