---
layout: single
title:  "Evaluation of GoMR Against Spark"
header:
categories: 
  - distributed systems
tags:
  - MapReduce
---

I created GoMR to solve a simple problem: make it efficient and painless to
deploy MapReduce jobs on a single, moderately powerfull machine. Despite not
having as pretty of an interface as today's competitor, Apache Spark, I believe
I was successful in achieving this goal. As well, the system and applications
are written in my favorite language, Go.

In this article, I compare the performance of GoMR against one of the
heavy-weights of the MapReduce world, Apache Spark.

# Experimental Setup
I compare GoMR to Spark with two programs, the canonical wordcount and a
triangle-counting program. As well, I perform the evaluation on two machines: my
6-core, 16GB desktop, and a 32-core, 200GB server. For brevity, I'll refer to
them as "desktop," and "server," respectively. Run-times are given in seconds.
Results are the average of 10 trials, unless otherwise stated.

The code for the evaluation can be found here:
- [Wordcount](https://github.com/cnnrznn/gomr/tree/master/examples/wordcount)
- [Counting Triangles](https://github.com/cnnrznn/gomr/tree/master/examples/count-triangles)

## Parallelism
To eliminate another variable in the evaluation, I have a single independent
variable, parallelism. Parallelism defines the number of mappers and reducers
allocated to a job, and they are the same. So $parallelism=1$ means one mapper,
one partitioner, and one reducer. $parallelism=2$ means two mappers, two
partitioners, and two reducers, and so on. Further experiments could be done to
capture trade-offs between more mappers than reducers and vice-versa. I'm sure
that work has been done before, so to keep things simple let's just have a
single number.

## Multiplex and Parallel
I include the evaluation for two version of GoMR: multiplex and parallel. The
multiplex uses `TextFileMultiplex` and parallel uses `TextFileParallel`. The
important results will be found by paying attention to the parallel versions of
the code. If not stated, assume the code is using `TextFileParallel`. Evaluating
wordcount using `TextFileMultiplex` is purely done for investigating Go's
channel performance under one reader and multiple writers, and I would not
recommend using that code in production.

# Wordcount
For this evaluation, I'll be using the following html document:
[moby dick](https://www.gutenberg.org/files/2701/2701-h/2701-h.htm). I have
replicated it to a size of 1.8 GB [here](files/gomr/moby.txt).

Let's take a look at the "hello-world" of MapReduce and see how a program
written in GoMR is different from it's Spark counterpart. A spark wordcount
program is typically defined as follows.

```python
sc = SparkContext("local[{}]".format(parallelism), "Wordcount")

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
    counts := make(map[string]int)

	for elem := range in {
		for _, word := range strings.Split(elem.(string), " ") {
                        counts[word]++
		}
	}

    for k, v := range counts {
        out <- Count{k, v}
    }

	close(out)
}

func (w *WordCount) Partition(in <-chan interface{}, outs []chan interface{}, wg *sync.WaitGroup) {
	for elem := range in {
		key := elem.(Count).Key

		h := sha1.New()
		h.Write([]byte(key))
		hash := int(binary.BigEndian.Uint64(h.Sum(nil)))
		if hash < 0 {
			hash = hash * -1
		}

		outs[hash%len(outs)] <- elem
	}

	wg.Done()
}

func (w *WordCount) Reduce(in <-chan interface{}, out chan<- interface{}, wg *sync.WaitGroup) {
	counts := make(map[string]int)

	for elem := range in {
		ct := elem.(Count)
		counts[ct.Key] += ct.Value
	}

	for k, v := range counts {
		out <- Count{k, v}
	}

	wg.Done()
}
```

## Server

 Paralellism | Spark | GoMR Multiplex | GoMR Parallel
---:|:---:|:---:|:---:
1  |733.46 |69.05 | 64.39
2  |378.04 |37.56 | 32.04
4  |201.77 |20.17 | 16.96
8  |111.56 |15.39 | 9.57
16 |67.89  |22.41 | 6.66
32 |62.38  |27.28 | 5.69

![Server Wordcount Performance](/images/evaluate-gomr/plot-wordcount-server.svg){: .align-center}

From this data I can make a few key observations.

First, Spark appears asymptotic for the 16 and 32 cases. This can be explained
by the fact that on a 32 core machine, 16 mappers and 16 reducers can be
scheduled at once. Upgrading to 32 mappers and reducers can't improve
performance as the tasks fight for hardware.

GoMR Multiplex runtime appears to be optimal at 8 cores, then go up again. My
suspicion is that because a single reader is supplying input to mapper channels
in round-robin fashion, increasing the number of mappers only causes an
increased waiting time at a fixed reading rate. One can imagine that as the
"ring-buffer" of mappers grows, it takes longer for the reader to make a
revolution and feed another input to a waiting mapper.

GoMR, in general, crushes spark. There are probably many sources of latency
Spark must incur that causes this. For example, Spark is made to be distributed
and fault tolerant, so communication likely happens - even on a single machine -
at the network level. A less major source of latency is likely the JVM.

I believe the algorithmic key to GoMR's success is the scheduling of the
mappers, combiners, and reducers. Because _all_ of the map and combine work is
done before the reducers start, the reducers and mappers aren't fighting for CPU
time. In Spark, the `reduceByKey` call is likely triggering a combine step that
is running concurrently to the reduce step. This creates additional contention
for resources and scheduling overhead.

## Desktop

 Paralellism | Spark | GoMR Multiplex | GoMR Parallel
---:|:---:|:---:|:---:
1 | 724.25 |55.18 | 46.22
2 | 376.13 |29.15 | 24.44
4 | 223.47 |15.59 | 12.98
8 | 182.33 |14.61 | 10.33

![Desktop Wordcount Performance](/images/evaluate-gomr/plot-wordcount-desktop.svg){: .align-center}

# A Note on Go Channels and MR

In the beginning, I wrote GoMR's map function to not perform a combine function.
That is, the mapper would simply pass each word to the appropriate reducer.
Because of this, I wound up with terrible performance.

![Bad Wordcount Algorithm](/images/evaluate-gomr/bad-algo.svg){: .align-center}

The reason for this was that I was sending every word in the file through shared
memory. [This talk](https://www.youtube.com/watch?v=KBZlN0izeiY) gives a good
explanation of how this was invoking the channel's locking mechanism. Clearly,
doing unnecessary communication was inefficient. Once I had augmented my
algorithm with a combiner, the picture changed:

![Good Wordcount Algorithm](/images/evaluate-gomr/plot-wordcount-server.svg){: .align-center}

Nice.
