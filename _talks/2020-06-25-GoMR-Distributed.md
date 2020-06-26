---
title: "GoMR Distributed"
collection: talks
type: "Tech Talk"
permalink: /talks/2020-06-25-GoMR-Distributed
venue: "Denver Mile High Gophers"
date: 2020-06-25
location: "Denver, CO, USA"
---

More information [here](/distributed%20systems/MapReduce-Framework-for-Golang/) and [here](/distributed%20systems/evaluation-of-gomr/).
<br>
[Slides](/files/GoMR/slides-distributed.pptx)
<br>
[Code](https://github.com/cnnrznn/gomr)

## Abstract

In the original GoMR talk, I presented my implementation of a local mapreduce
framework for Golang. In this talk, I give the same overview of mapreduce and
the implementation, and show how I scale the system using Kubernetes.

## Description

Frustrated trying to run/debug MapReduce applications on my laptop, I decided
to spin my own MR framework to solve the problems of efficiency,
configuration, and error messages. Go is a compiled language and has inherent
support for distribution in the form of channels. I took advantage of these
two strengths to create GoMR.

In this talk, I give an overview of the framework, the programming style I
used to create it, and an evaluation of the framework against a current state
of the art system, Apache Spark.

I show how I leveraged Kubernetes, a container orchestration system, to scale
GoMR to many machines. As much as possible, GoMR off-loads the work of a
control plane onto Kubernetes.

A more complete description can be found
[here](https://connorzanin.com/distributed%20systems/MapReduce-Framework-for-Golang/),
and its evaluation
[here](https://connorzanin.com/distributed%20systems/evaluation-of-gomr/).

I hope this work will encourage a move away from the cumbersome JVM, and spark
a next-generation of efficient distributed compute frameworks.