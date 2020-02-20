---
layout: single
title:  "Draft Post"
header:
  teaser: "unsplash-gallery-image-2-th.jpg"
categories: 
  - Jekyll
tags:
  - edge case
---
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
