---
layout: single
title:  "GoMR revisited"
---

A few months ago, I quit my job.
Since then one of my goals has been to clean up my long running MapReduce framework, GoMR.
Originally, GoMR was hacked together quickly in preparation for a talk I gave at the Denver Gophers Meetup.
Because of this, many corners were cut that I believe prevented the system and the software from being being something I was truly proud of.

And, this summer, I have blown up the repository and rebuilt GoMR from its foundation.
The project is in a place I feel comfortable leaving it.
I believe the software has a clean architecture, and when I decide to implement a control plane, could be production ready.

In this article, I want to discuss what was wrong with the old system, and what is right with this one.
I want to talk about software architecture, separation of concerns, and problem isolation.
I want to talk about simplicity.

# What is MapReduce?

MapReduce is a method for distributing computation on large data sets about a cluster of machines.
The concept was made public by fabled giga-engineer Jeff Dean in his seminal paper, _MapReduce: simplified data processing on large clusters_.
In brief, MapReduce recognizes that there are two fundamental stages to data processing: transformation and reduction.
Transformations are operations performed on data that are not dependent on other data.
For example, if you have a set of numbers and want to find the square of each number, each operation can be done independently on different machines.
Reductions, on the other hand, are operations that depend on the data being reduced.
In the canonical example wordcount, words are read from text, parsed, and reduced to determine a count for each word.
The computation of counting words depends on seeing all instances of the word being counted.

MapReduce recognizes that all "big data" computations can be defined by a series of such pairs - map and reduce - of these computations.
In each "stage," data is transformed, then shuffled to group data by its key for the reduce step, then reduced to a resulting value.
A system that implements MapReduce provides the framework for executing the computation, performing the shuffle step (the most complex and interesting part IMO), and providing a result to the user.

# What has changed

## The old

To give my original talk on the GoMR system, I wanted to collect data comparing Spark to GoMR.
And, to really flex, I wanted to collect data from the system running on a cluster, not just a single node.
In the end, this led to the distributed version of my system being coupled - not highly, completely - to Kubernetes.
This coupling meant that the GoMR repository was doing two jobs simultaneously: resource management and data processing.
For the new version, I only wanted to focus on the latter.
I don't want to control how people deploy and run each executable instance beyond reasonable basic assumptions.

For those interested, the code lives under a tag at [v0.2](https://github.com/cnnrznn/gomr/releases/tag/v0.2).

## The new

Now, with the time to invest in building the system the way I truly wanted, I decided on a few design points:

1. **No coupling or dependency on a cluster orchestrator**: Give the user a binary and let them decide how to run it
2. **Keep the library simple**: only focus on data plane
3. **Flexibility and simplicity over LOC**: By not making assumptions for the user, the library stays small and easy to understand.

These three principles drove the development of this iteration of GoMR.
Ultimately, I recognize that this may ever only be used by me, but I am hoping it can serve as a learning tool for others and therefore simplicity I hold above all else.

The new GoMR is a library, meant to be imported and built into a binary that will run the user's code on a cluster - more on architecture later.

# Developer experience - Usage

- GoMR requires more boilerplate than other systems.
- This boilerplate (1) is teachable (exposes internals of mapreduce) and (2) allows flexibility
- A Mapper and Reducer: two functions that consume data from a channel, transform the data, and write the result to a channel
- Two Data types. Input and intermediate data

The library's API is defined in `interface.go`

# Architecture

GoMR has a three tiered architecture (4 if you include user code).
Each tier is defined by what it is responsible for executing - what's below it - and what results it provides to the tier above it.
I will explain each tier in these terms.

The first tier is responsible for invoking user code.
It passes data through channels to the mappers and reducers, and reads the results from their output channels.
To do this, the first tier manages go routines and channels.
For example, when a tier 1 reducer receives a piece of data with a new reduce key, it

## Tiers
