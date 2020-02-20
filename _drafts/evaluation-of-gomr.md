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
6-core, 16GB desktop, and a 32-core, 200GB server.

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

## Wordcount
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

| Parallelism | Spark | GoMR-Multiplex | GoMR-Parallel |
|---|-----|------|
|||||
