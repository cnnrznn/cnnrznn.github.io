---
layout: single
title:  "A Reddit-Style Comment System in Golang"
header:
  image: "inverted-tree.gif"
categories: 
  - app-dev
---

Preparing for job interviews, I found this question on Glassdoor. The problem
was to design a commenting system similar to Reddit's. Here, I use this problem
to show how to problem solve using Golang's ease of HTTP/API development. I will
present my Golang implementation which is based on grpc. Also, I'll review the
database design (MySQL).

## Source Code
The complete source code for this project can be found
[here](https://github.com/cnnrznn/CommentSystem).

## Features

![A Reddit comment](/images/comment/reddit-comment.PNG)

Looking at Reddit's user interface, there are a few design features that stand
out. The first and most prominent is the nesting of comments. Second, there is
the voting system. Third, there is the "masking" of comments. Some comments are
displayed while others are minimized.

In this post I will consider the design of the first two features. Deciding
which comments to "shrivel" will remain outside the scope of this post.

## Tools
For this program I will be writing the majority of the code in Golang. In the
github repository, you can find a basic javascript client that does nothing but
pretty-print the comment tree as a JSON. I will connect the server code to a
MySQL database through Golang's `database/sql` library.

## API
Let's first design the API for the commenting service. To fulfill our basic
needs, the comment system needs two features:

- `/Comment/New` - to create a new comment
- `/Comment/List` - to display comments

There is another endpoint, `/Comment/Delete`, which we will not cover here.
However, if we were to implement this we could simply change the text of the
comment to "[deleted]" as it is done on Reddit. It is critical to keep the
comment to retain the structure of the comment tree, so a delete feature will
most likely not destroy all traces of the comment.

### Implementation

To implement an API server in Go, w can use the `net/http` package which
provides a server multiplexer,
[ServeMux](https://golang.org/pkg/net/http/#ServeMux). Next, for every endpoint
we wish to serve, we must provide the muxer with a mapping of path to a
[Handler](https://golang.org/pkg/net/http/#Handler) object. While this _could_
be an empty struct, I want to keep a cached view of the comment tree in server
memory. So, my server object will have a data structure to do just that:

```go
type CommentServer struct {
    tree map[int]*comment.Comment
}
```

We then create a ServeMux object and redirect http requests to our server
object:

```go
cs := CommentServer{}

mux := http.NewServeMux()
mux.Handle("/Comment/New", cs)
mux.Handle("/Comment/List", cs)

// Run the server
log.Fatal(http.ListAndServe(":8888", mux))
```

Finally, we create code to handle an http request for the server object (which
implements the `Handler` interface.

```go
func (s *CommentServer) ServeHttp(w http.ResponseWriter, req *http.Request) {
    switch req.URL.Path {
    case "/Comment/New":
        // handle new
    case "/Comment/List":
        // handle list
    }
}
```

