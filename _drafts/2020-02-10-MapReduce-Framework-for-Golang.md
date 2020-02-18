---
layout: single
title:  "GoMR: A MapReduce Framework for Golang"
categories: 
  - distributed systems
tags:
  - batch processing
  - mapreduce
---

In a world of big data and batch processing, MapReduce is unavoidable. But my
recent experience of getting Hadoop up and running for single-node debugging was
a nightmare. Here, I present my implementation of the MapReduce framework
written in my favorite language, Go, and designed for a single machine.

The code can be found [here](https://github.com/cnnrznn/gomr).

# Abstract

MapReduce was created to solve the problem of running large computations accross
many low-powered, cheap machines. However, today's consumer machines are much
more powerful. It is not uncommon for a consumer laptop to have 8 or more CPUs
and 16+ GB of RAM. With machines of this capability, surely the types of
computations we can do on a single machine have changed.

In this article, I present the design, implementation, and evaluation of a new
MapReduce framework, ***GoMR***. GoMR is designed for moderate to high-power
consumer and server machines that can have the memory capacity to run mapreduce
jobs locally.  GoMR makes it easy to write map-reduce code that instantly scales
to efficiently use the resources of a single machine.

# Motivation

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

# Design

The design of the one-machine MapReduce is quite simple. The _library_ will define
some types for mappers, reducers, and partitioners. It will also take
care of "architecting" the connections between all of these components and
synchronizing stages of the system. By architecting, I mean the engine first
allocates Go channels. Then, it spawns the map, partition, and reduce
functions all with the correct channels linking them.

The _driver_ is a program the user of the library will write. The driver is
responsible for handling input and output values from the map and reduce stages,
respectively.

## Combiners?

The observant reader will notice I have ignored the use of combiners. Since this
is a framework for a single machine, having a combiner does not make sense. In
distributed MapReduce, combiners function to limit the ammount of data sent over
the network. Here, there is no such optimization possible. In fact, a combiner
would only add overhead as more channels would need to be created to connect the
combiner to the mapper and partitioner.

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
gomr.TextFile(fn, inMaps)
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

# Evaluation

Here I present a comparison of GoMR against Apache Spark. I ran these
experiments on a 6-core AMD machine with 16GB of memory.

The framework comes with some
[examples](https://github.com/cnnrznn/gomr/tree/master/examples). For now, they
include the canonical wordcount, a 2-cycle counting program, and a triangle
counting program.

## Wordcount
For this evaluation, I'll be using the following html document:
[moby dick](https://www.gutenberg.org/files/2701/2701-h/2701-h.htm). I have
replicated it to a size of 1.8 GB.

Let's take a look at the "hello-world" of MapReduce and see how a program
written in GoMR is different from it's Spark counterpart. A spark wordcount
program is typically defined as follows.

```python
sc = SparkContext("local", "Wordcount")

text_file = sc.textFile(sys.argv[1])

counts = text_file.flatMap(lambda line: line.split(" ")) \
                  .map(lambda word: (word, 1)) \
                  .reduceByKey(lambda a, b: a + b)

for count in counts.collect():
    print(count)
```

Unfortunately, the wordcount in Go comes out much longer at about 75 lines. The
three important functions are given in the following snippet.

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

| Nodes | System | Time |
|---|-----|------|
|1| Spark | 11m 41.550s |
|1| **GoMR** | 8m 59.886s |
|2| Spark | 6m 30.589s |
|2| **GoMR** | 6m 26.743s |
|4| Spark | 3m 45.6s |
|4| **GoMR** | 4m 25.9s  |
|8| Spark | 3m 2.8s |
|8| **GoMR** | 3m 21.8s  |

Interestingly, GoMR initially beats Spark, but then falls behind as we increase
the number of nodes. This is due to the behavior of the Go file object that has
a parallel read method. More on this is in the discussion. However, when I
increase the parallelism, Spark takes the lead. This is because Spark is able to
read files in parallel, making better use of the machine's available resources.
However, for low parallelism the reader is restricted. This is why GoMR wins in
the 1 and 2 node cases.

As discussed in the design, this inspired the creation of my own parallel read
function for files. Using this method, the performance numbers change a bit:

| Nodes | System | Time |
|---|-----|------|
|1| Spark       | |
|1| **GoMR**    | |
|2| Spark       | |
|2| **GoMR**    | |
|4| Spark       | |
|4| **GoMR**    | |
|8| Spark       | |
|8| **GoMR**    | |

# Discussion

During the evaluation I noticed some odd behavior from Spark and Go that I
thought is worth discussing.

## Golang's canonical `readFile()`

Go's file seems to implement a parallel Reader interface. I noticed when
scanning through the input the file, all available cores were being used.
Somewhere between file `Open()` and `Scan()`, the operations are being
parallelized. Unfortunately, I might be bottlenecking these operations by
funneling them through a single channel.