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
having as pretty of an interface as today's competitor, Apache Spark, I
believe I was successful in achieving this goal. As well, the system and
applications are written in my favorite language, Go.

In this article, I compare the performance of GoMR against one of the
heavy-weights of the MapReduce world, Apache Spark.

# Experimental Setup
I compare GoMR to Spark with two programs, the canonical wordcount and a
triangle-counting program. As well, I perform the evaluation on two machines:
my 6-core, 16GB desktop, and a 32-core, 200GB server. For brevity, I'll refer
to them as "desktop," and "server," respectively. Run-times are given in
seconds. Results are the average of 10 trials.

The code for the evaluation can be found here:
- [Wordcount](https://github.com/cnnrznn/gomr/tree/master/examples/wordcount)
- [Counting Triangles](https://github.com/cnnrznn/gomr/tree/master/examples/count-triangles)

## Parallelism
To eliminate another variable in the evaluation, I have a single independent
variable, parallelism. Parallelism defines the number of mappers and reducers
allocated to a job, and they are the same. So $parallelism=1$ means one
mapper, one partitioner, and one reducer. $parallelism=2$ means two mappers,
two partitioners, and two reducers, and so on. Further experiments could be
done to capture trade-offs between more mappers than reducers and vice-versa.
I'm sure that work has been done before, so to keep things simple let's just
have a single number.

## Multiplex and Parallel
I include the evaluation for two version of GoMR: multiplex and parallel. The
multiplex uses `TextFileMultiplex` and parallel uses `TextFileParallel`. The
important results will be found by paying attention to the parallel versions
of the code. If not stated, assume the code is using `TextFileParallel`.
Evaluating wordcount using `TextFileMultiplex` is purely done for
investigating Go's channel performance under one reader and multiple writers,
and I would not recommend using that code in production.

## Scope and Hyperparameters
For simplicitly, in this evaluation I am comparing the default
hyper-parameters of Spark and GoMR. That is, I use the default configurations
for each framework, and only vary the $parallelism$ variable discussed above.

For GoMR, the hyperparameters would be the amount of buffering on channels
and the amount of data read from the input file at a time. The defaults for
these are 4096 and 32 MB, respectively.

For Spark there are [many many
parameters](https://spark.apache.org/docs/latest/configuration.html).

# Wordcount
For this evaluation, I'll be using the following html document:
[moby dick](https://www.gutenberg.org/files/2701/2701-h/2701-h.htm). I have
replicated it to a size of 1.8 GB.

**Caveat:** When running the experiments, I accidentally used a different size
input file on the server and desktop. They are both *around* 1.8 GB, but they
differ slightly. This is why the results for the server and client are
inconsistent by a significant margin. However, the input file is consistent
across experiments on the same machine. All server experiments use the same
input, and all desktop experiments use the same input.

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

In the beginning, I wrote GoMR's map function to not perform a combine
function. That is, the mapper would simply pass each word to the appropriate
reducer. Because of this, I wound up with terrible performance.

![Bad Wordcount Algorithm](/images/evaluate-gomr/bad-algo.svg){: .align-center}

The reason for this was that I was sending every word in the file through
shared memory. [This talk](https://www.youtube.com/watch?v=KBZlN0izeiY) gives
a good explanation of how this was invoking the channel's locking mechanism.
Clearly, doing unnecessary communication was inefficient. Once I had
augmented my algorithm with a combiner, the picture changed:

![Good Wordcount Algorithm](/images/evaluate-gomr/plot-wordcount-server.svg){: .align-center}

Nice.

# Counting Triangles

Counting triangles is a graph analysis algorithm where the objective is to
find instances of 3 nodes that form a clique. For the purpose of this
evaluation, we will only focus on counting the number of triangles. If we had
a machine with ample disk space, we could also flush these entries to disk.

The input for this evaluation is [edges.csv](https://drive.google.com/file/d/1hnzD93AE3H2nhp5-RSuRCmMRoIJS_NzR/view?usp=sharing), a
file representing a twitter graph. Unfortunately, the original source of the
data is no longer available. It seems [the
website](http://socialcomputing.asu.edu/datasets/Twitter) has been taken
down.

Recall that a motivation for this work is my grief with JVM memory
allocation. I can run the GoMR implementation on the full dataset, but
haven't been able to wrangle the Spark implementation to do the same. To that
end, I implemented a filter to remove any edges to and from vertices above a
threshold. I adjusted this threshold so that the experiment runs for enough
time but does not exceed the jvm limitation.

## Implementations

- [GoMR](https://github.com/cnnrznn/gomr/blob/master/examples/count-triangles/triangles.go)
- [PySpark](https://github.com/cnnrznn/gomr/blob/master/examples/count-triangles/triangles.py)

## Server

![Server Trianglecount Performance](/images/evaluate-gomr/plot-trianglecount-server.svg){: .align-center}

 Paralellism | Spark | GoMR
---:|:---:|:---:
1 | 1688.13 | 230.39
2 | 879.44 | 134.17
4 | 473.32 | 71.13
8 | 267.71 | 41.13
16 | 177.90 | 31.68
32 | 152.49 | 20.11

## Laptop
***Note*** due to Coronavirus, I am away from my apartment and unable to
execute the experiment on my desktop machine. Instead, these experiments were
run on my laptop which has 8 cores (intel core i7) and 16 GB of memory with
48G of swap. The tests are run on windows subsystem for linux (WSL).

![Laptop trianglecount Performance](/images/evaluate-gomr/plot-trianglecount-laptop.svg){: .align-center}

 Paralellism | Spark | GoMR
---:|:---:|:---:
1 | 932.25 | 198.20
2 | 499.75 | 106.15
4 | 313.85 | 60.09
8 | 281.04 | 44.80
16 | 282.04 | 46.38
32 | 293.55 | 45.11 
