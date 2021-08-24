---
title: "The Raft Consensus Algorithm and Go"
collection: talks
type: "Talk"
permalink: /talks/Go-Raft
venue: "Mile High Gophers"
#date: 2021-09-22
location: "Denver, Colorado"
published: true
---


Go is a great language for  building local distributed systems.
With channels and goroutines, it is straightforward to distribute work across many workers on a single machine.
But what if we want to take advantage of multiple machines?
While goroutines make it easy to communicate between processes on the same machine, they don't solve the problem of coordination _across_ machines.
In distributed systems, the most common problem is consensus: how can we get all participants to agree on something?
Raft is an algorithm that solves the consensus problem.


In this talk, I give an overview of Raft as well as some distributed system fundamentals.
As well, I demonstrate an implementation of a consensus protocol in the Go programming language.
The goal of this talk is to teach concepts that will help the audience design and build better systems.
It is also to showcase some Go code :)
