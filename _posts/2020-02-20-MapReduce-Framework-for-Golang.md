---
layout: single
title:  "GoMR: A MapReduce Framework for Golang"
categories: 
  - distributed systems
tags:
  - batch processing
  - MapReduce
---

In a world of big data and batch processing, MapReduce is unavoidable. But my
recent experience of getting Hadoop up and running for single-node debugging was
a nightmare. Here, I present my implementation of the MapReduce framework
written in my favorite language, Go, and designed for a single machine.

The code can be found [here](https://github.com/cnnrznn/gomr).

# Abstract

MapReduce was created to solve the problem of running large computations across
many low-powered, cheap machines. However, today's consumer machines are much
more powerful. It is not uncommon for a consumer laptop to have 8 or more CPUs
and 16+ GB of RAM. With machines of this capability, surely the types of
computations we can do on a single machine have changed.

In this article, I present the design, implementation, and evaluation of a new
MapReduce framework, ***GoMR***. GoMR is designed for moderate to high-power
consumer and server machines that can have the memory capacity to run MapReduce
jobs locally.  GoMR makes it easy to write map-reduce code that instantly scales
to efficiently use the resources of a single machine.

# Motivation

MapReduce frameworks are great because they abstract away all of the nasty
details of distributed infrastructure. The framework provides the programmer
with the capability to distribute their job over many machines, and the
programmer provides the framework the computation logic. Swell.

Recent experience had left me banging my head against a wall trying to deploy
these frameworks on a single machine for debugging. In the case of Hadoop, I
either (1) could run in stand-alone mode but with no way of increasing the number
of cores and memory the system would use, or (2) run in pseudo-stand-alone mode
with the same problems.

I love Go and wanted to ditch Java for Golang. The abstraction of channels, the
runtime's awareness of the number of available cores, and not having to figure
out how to set an upper/lower limit for the jvm's heap size were all
motivations for doing this.

# Design

The design of the one-machine MapReduce is quite simple. The _library_ will define
some types for mappers, reducers, and partitioners. It will also take
care of "architecting" the connections between all of these components and
synchronizing stages of the system. By architecting, I mean the engine first
allocates Go channels. Then, it spawns the map, partition, and reduce
functions all with the correct channels linking them.

The _driver_ is a program the user of the library will write. The driver is
responsible for handling input and output values from the map and reduce stages,
respectively. It is also responsible for defining the logic of the MapReduce
job.

## Channels or Values?

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

## User API

To use the framework, the user must implement a `Mapper`, `Partitioner`, and
`Reducer`. The library's `Run()` method builds a MapReduce architecture with the
supplied number of mappers and reducers.

```go
func Run(nMap, nRed int, m Mapper, p Partitioner, r Reducer) (inMap []chan interface{},
                                                              outRed chan interface{})
```

The library returns two channels, `inMap` and `outRed` for supplying input
values and delivering outputs from the reduce.

The interfaces for a mapper, reducer and partitioner are as follows.

```go
type Mapper interface {
    Map(in <-chan interface{}, out chan<- interface{})
}

type Partitioner interface {
    Partition(in <-chan interface{}, outs []chan interface{}, wg *sync.WaitGroup)
}

type Reducer interface {
    Reduce(in <-chan interface{}, out chan<- interface{}, wg *sync.WaitGroup)
}
```

## Combiners?

The observant reader will notice I have ignored the use of combiners. Since this
is a framework for a single machine, having a separate entity for a combiner
does not make sense. In distributed MapReduce, combiners function to limit the
amount of data sent over the network. In GoMR, we would not want to create more
channels to send data from the mappers to the combiners as this is a waste of
precious single-machine resources.

Instead, combiner logic in GoMR should exist as a part of the mapper function.
For example, we could write a wordcount  map function like so.

```go
func (w *WordCount) Map(in <-chan interface{}, out chan<- interface{}) {
    counts := make(map[string]int)

    for elem := range inMap {
        for _, word := range strings.Split(elem.(string), " ") {
            counts[word]++
        }
    }

    for k, v := range counts {
        out <- Count{k, v}
    }

    close(out)
}
```

As stated before, this does introduce a higher burden on the user of the GoMR
library. However, since the goal of this project is efficiency over simplicity,
we will accept this optimization.

## Input
After doing a first round of evaluation, I suspected that file input to the
mappers may be a bottleneck. In the beginning, I created a single input channel
and had all mappers read from that channel. While this is legal in Go and
produces the correct result, I wondered if I could do better. My first
implementation was something like the following.

```go
// Library-side
inMap := make(chan interface{}, nMap * CHANBUF)
```
```go
// Driver-side
scanner := bufio.NewScanner(file)
for scanner.Scan() {
    inMap <- scanner.Text()
}
close(inMap)
```

I decided to follow this suspicion and created a second version. This time, I
would create a separate input channel for every mapper and multiplex the input
among them:

```go
// Library-side
inMaps := make([]chan interface{}, nMap)
for i:=0; i<nMap; i++ {
    inMaps[i] = make(chan interface{}, CHANBUF)
}
```
```go
// Driver-side
gomr.TextFileMultiplex(fn, inMaps)
```
```go
// Library - TextFileMultiplex()
scanner := bufio.NewScanner(file)
for d:=0; scanner.Scan(); d=(d+1)%nMap {
    inMaps[d] <- scanner.Text()
}
for i:=0; i<nMap; i++ {
    close(inMaps[i])
}
```

Of course, there is still one more optimization I could make. Instead of having
a single reader multiplex among the input channels, I could have multiple
readers each seek to a section of the file and feed their input to their
respective mapper.

This makes the implementation significantly more complicated. Now, we have to
chunk the file both by size and by newline, making sure no input lines are
skipped or duplicated in the computation.

```go
func TextFileParallel(fn string, inMap []chan interface{}) {
	file, err := os.Open(fn)
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	stat, err := file.Stat()
	if err != nil {
		log.Fatal(err)
	}

	size := stat.Size()
	nChunks := len(inMap)
	chunkSize := int64(math.Ceil(float64(size) / float64(nChunks)))

	for i := 0; i < nChunks; i++ {
		go func(i int) {
			buffer := make([]byte, FILEBUF)
			atEOF := false
			skippedFirst := false

			start := chunkSize * int64(i)
			end := start + chunkSize
			bufstart, bufend := 0, 0
			log.Println(i, start, end)

			file, _ := os.Open(fn)
			defer file.Close()

			pos, err := file.Seek(start, 0)
			if err != nil || pos != start {
				log.Fatal(pos, err)
			}

			for start <= end && !atEOF {
				copy(buffer, buffer[bufstart:bufend])
				bufend -= bufstart

				n, err := file.Read(buffer[bufend:])
				if err != nil {
					if err == io.EOF {
						atEOF = true
					} else {
						log.Fatal(err)
					}
				}

				bufstart = 0
				bufend += n

				for start <= end {
					advance, token, err := bufio.ScanLines(buffer[bufstart:bufend], atEOF)
					if err != nil {
						log.Fatal(err)
					}

					if advance == 0 {
						break
					}

					bufstart += advance
					start += int64(advance)

					if i == 0 || skippedFirst {
						inMap[i] <- string(token)
					}
					skippedFirst = true
				}
			}

			close(inMap[i])
		}(i)
	}
}
```

# Examples
## Wordcount
Of course, we need to start with the canonical wordcount example. The code can
be found [here](https://github.com/cnnrznn/gomr/tree/master/examples/wordcount).
This program has two versions for evaluation purposes. The difference between
the two is which library function they use for input.

```go
func (w *WordCount) Map(in <-chan interface{}, out chan<- interface{}) {
	for elem := range in {
		for _, word := range strings.Split(elem.(string), " ") {
			out <- word
		}
	}

	close(out)
}

func (w *WordCount) Partition(in <-chan interface{}, outs []chan interface{}, wg *sync.WaitGroup) {
	for elem := range in {
		key := elem.(string)

		h := sha1.New()
		h.Write([]byte(key))
		hash := int(binary.BigEndian.Uint64(h.Sum(nil)))
		if hash < 0 {
			hash = hash * -1
		}

		outs[hash%len(outs)] <- key
	}

	wg.Done()
}

func (w *WordCount) Reduce(in <-chan interface{}, out chan<- interface{}, wg *sync.WaitGroup) {
	counts := make(map[string]int)

	for elem := range in {
		key := elem.(string)
		counts[key]++
	}

	for k, v := range counts {
		out <- Count{k, v}
	}

	wg.Done()
}

```

## Counting Triangles
The reason I started this project was this example, found
[here](https://github.com/cnnrznn/gomr/tree/master/examples/count-triangles).
This program takes an edge file where every line is two vertex id's separated by
a comma, and calculates the number of triangles in the graph. A triangle is
defined as three unique vertices, A, B, and C, with edges A->B->C->A.

```go
type EdgeToTables struct {
	edges map[Edge]bool
}

func (e *EdgeToTables) Map(in <-chan interface{}, out chan<- interface{}) {
	for elem := range in {
		edge := elem.(Edge)
		if edge.Fr < edge.To {
			out <- JoinEdge{edge.To, "e1", edge}
		}
		out <- JoinEdge{edge.Fr, "e2", edge}
	}

	close(out)
}

func (e *EdgeToTables) Partition(in <-chan interface{}, outs []chan interface{}, wg *sync.WaitGroup) {
	for elem := range in {
		je := elem.(JoinEdge)
		outs[je.Key%len(outs)] <- je
	}

	wg.Done()
}

func (e *EdgeToTables) Reduce(in <-chan interface{}, out chan<- interface{}, wg *sync.WaitGroup) {
	jes := []JoinEdge{}

	for elem := range in {
		je := elem.(JoinEdge)
		jes = append(jes, je)
	}

	log.Println("Begin sorting")
	sort.Sort(ByKeyThenTable(jes))
	log.Println("End sorting")

	numTriangles := 0
	lastSeen := -1
	arr := []Edge{}

	for _, je := range jes {
		if je.Key != lastSeen {
			arr = nil
			lastSeen = je.Key
		}

		if je.Table == "e1" {
			arr = append(arr, je.Edge)
		} else {
			for _, e1 := range arr {
				if e1.Fr < je.Edge.To {
					if _, ok := e.edges[Edge{je.Edge.To, e1.Fr}]; ok {
						numTriangles++
					}
				}
			}
		}
	}

	out <- numTriangles
	wg.Done()
}
```

# Conclusion
With this article, I showed how one can easily spin their own MapReduce
framework. I presented my implementation in Go and gave some examples of
programs I wrote using the framework.

In my next post, I will present the evaluation of GoMR against a popular, mature
MR framework, Spark.
