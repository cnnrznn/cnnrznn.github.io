---
layout: single
title:  "Time, Clocks, and Leslie Lamport"
categories: 
  - distributed systems
---

In the paper _Time, Clocks, and the Ordering of Events in a Distributed System, Leslie Lamport defines the _happens before_ relationship for distributed systems.
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

