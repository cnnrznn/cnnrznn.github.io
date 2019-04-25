---
layout: single
title:  "Partial Synchrony"
tags:
  - distributed systems
---

In this article, I give an overview and discussion of the partial synchrony
network model. In distributed systems, it is important to consider the
assumptions about the network. Many systems today are designed and built based
on the partial synchrony model which makes it crucial to understand.

## Introduction
Distributed systems are designed to do some *computation* within some
*environment*. When designing a system it is necessary to know the assumptions
that are made about the environment. For example:

* What is the model of computation?
* What is the  connectivity of the network?
* What is the threat model?
* What is the failure model for processes?
* What is the failure model for the network?

Answers to these questions determine the requirements for a solution to the
desired computation. One solution under a set of assumptions may not hold under
a different set. For example, a solution to the consensus problem under the
benign failure model will be very different from a solution under the byzantine
failure model.

We also make assumptions about how the network delivers messages. On one end of
the spectrum, we could assume a *synchronous* network. That is that all messages
are received within some fixed and known upper-bound $\Delta$ from when they
were sent. On the other end is the *asynchronous* network. An asynchronous
network may impose arbitrary delays on messages. In other words, $\Delta$ is
neither fixed nor known.
