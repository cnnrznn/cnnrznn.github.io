---
layout: single
title:  "GoMR: A MapReduce Framework for Golang"
categories: 
  - distributed systems
---

In a world of big data and batch processing, MapReduce is unavoidable. But my
recent experience of getting Hadoop up and running for single-node debugging has
been a nightmare. Here, I present my implementation of the MapReduce framework
written in my favorite language and designed for a single machine.

The code can be found [here](https://github.com/cnnrznn/gomr).

## The Motivation

MapReduce frameworks are great because they abstract away all of the nasty
details of distributed infrastructure. The framework provides the programmer
with the capability to distribute their job over many machines, and the
programmer provides the framework the computation logic. Swell.

Recent experience had left me banging my head against a wall trying to deploy
these frameworks on a single machine for debugging. In the case of Hadoop, I
either (1) could run in standalone mode but with no way of increasing the number
of cores and memory the system would use, or (2) run in pseudo-standalone mode
with the same problems.

I love Go and wanted to ditch Java for Golang. The abstraction of channels, the
runtime's awareness of the number of available cores, and not having to figure
out how to set an upper/lower limit for the jvm's heap size were all
motivations for doing this.

## The Design

The design of the one-machine MapReduce is quite simple. The _library_ will define
some types for mappers, reducers, and partitioners. It will also take
care of "architecting" the connections between all of these components and
synchronizing stages of the system. By architecting, I mean the engine first
allocates Go channels. Then, it spawns the map, partition, and reduce
functions all with the correct channels linking them.

The _driver_ is a program the user of the library will write. The driver is
responsible for handling input and output values from the map and reduce stages,
respectively.

### Combiners?

The observant reader will notice I have ignored the use of combiners. Since this
is a framework for a single machine, having a combiner does not make sense. In
distributed MapReduce, combiners function to limit the ammount of data sent over
the network. Here, there is no such optimization possible. In fact, a combiner
would only add overhead as more channels would need to be created to connect the
combiner to the mapper and partitioner.

### Channels or Values?

A design decision that came up in this process was the decision to have the user
deal with channels or values. For example, we could supply either of the
following interfaces for a map function:

```go
type Mapper interface {
    Map(input interface{}) interface{}
}
```
or
```go
type Mapper interface {
    Map(in <-chan interface{}, out chan<- interface{})
}
```

What complexity should we expose to the user? In the first case, the user would
only have to deal with values. This is simpler than iterating a channel.
On the other hand, giving the user direct access to the channel allows them to
insert their own **initialization** and **cleanup** logic. For example, what if
the user wants to collect some aggregate statistics about their data. This
becomes harder if their function only exposes values. Sure, they could create
fields in the struct that implements the `Mapper` interface, but then they have
to deal with synchronization and locking on a Mapper-to-Mapper level. In the
end, I decided to expose more complexity and flexibility to the user.

### User API

To use the framework, the user must implement a `Mapper`, `Partitioner`, and
`Reducer`. The library's `Run()` method builds a MapReduce architecture with the
supplied number of mappers and reducers.

```go
func Run(nMap, nRed int, m Mapper, p Partitioner, r Reducer) (inMap, outRed chan interface{})
```

The library returns two channels, `inMap` and `outRed` for supplying input
values and delivering outputs from the reduce.


## Examples

The framework comes with some
[examples](https://github.com/cnnrznn/gomr/tree/master/examples). For now, they
include the canonical wordcount, a 2-cycle counting program, and a triangle
counting program.

### Wordcount
Let's take a look at the wordcount application and see how it's different from
the typical approach. A spark wordcount program is typically defined as follows.

```python
counts = text_file.flatMap(lambda line: line.split(" ")) \
                  .map(lambda word: (word, 1)) \
                  .reduceByKey(lambda a, b: a + b)
```
