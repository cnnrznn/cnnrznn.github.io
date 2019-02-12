---
layout: single
title:  "Byzantine Fault Tolerant Broadcast"
tags:
  - distributed systems
  - coding
---

An essential primitive in distributed systems is _broadcast_, or the ability to send a group of participants the same message.
In the honest case, this problem is straightforward; send everyone the message and acknowledge its receipt.
However, this problem becomes complicated under the assumption of byzantine faults.
If we apply the honest protocol to the byzantine environment, we may have different peers accept different values!
In this article, I give a summary and implementation of an algorithm for overcoming the problem.

## The Problem
The problem we are trying to solve is that of _reliable broadcast_.
Specifically, a broadcast protocol that guarantees all process accept the same value (or none) as the result of a broadcast within a closed group.

In this instance we consider the _byzantine fault model_.
That is, participants in our distributed system can behave arbitrarily.
Our only assumption is that _honest_ peers obey the protocol.

The other type of failure model usually considered is that of the network.
That is out of scope for this article.
We will simply assume here that messages from honest peers are delivered eventually.
Later, I will show how the implementation copes with messages from dishonest (byzantine) peers.
Then we will see that exactly-once delivery of messages from honest peers is nice, but not required.

## The Code
If you would like to follow along during this post, the code can be found at https://github.com/cnnrznn/clj-net.git.
The code for this post is located in the `src/clj_net/broadcast.clj` file.
The project is written in Clojure which is a beautiful functional language that is also practical as it runs on the JVM and has access to the Java ecosystem.
If you wish to run the code, you'll need [Leiningen](https://leiningen.org/).

## The Solution
The solution to the broadcast problem is to insert another layer in the stack: a reliable broadcast primitive.
The goal of the primitive is to reduce the power of byzantine processes to that of fail-stop processes.
Two properties must hold for a process _p_ broadcasting message *v*:

1. **Validity** If *p* is correct, all correct processes `Accept` *v*
2. **Agreement** If *p* is faulty, then either all correct processes agree on the same value, or no value is accepted from *p*

This protocol comes from an old paper by Gabriel Bracha _[Asynchronous Byzantine Agreement Protocols](https://core.ac.uk/download/pdf/82523202.pdf)_.
After establishing this primitive, Bracha uses it to achieve a randomized consensus algorithm resilient to *n >= 3f+1* faults!

The protocol has 3 phases:
1. The initiator broadcasts its message to all peers
2. The peers echo this message to indicate what they *think* the value is
3. The peers send ready messages to indicate a commit

### Message Format
The messages in the system have four components
* `owner`
* `sender`
* `round`
* `value`

The owner of the message is the process that initiated the broadcast protocol.
The sender indicates from whom the current message was sent.
The round is a number used to totally order broadcasts from a single process.
The value is the proposed result of the broadcast.

It should be noted that in a production environment, messages should be authenticated with cryptographic signatures.
For this implementation, it is sufficient for demonstration and testing to assign messages the `owner` and `sender` fields as integers.

### Message Filtering
In order to make a decision, each process must accept a number of other messages.
Under the byzantine fault model, typically a process waits for `2f+1` messages.
Recall that the maximum number of faulty processes in a consensus system is `n >= 3f+1`.
Therefore, we are guaranteed a _minimum_ of `2f+1` messages arriving, and must progress after they do.

But which messages do we wait for?
In the `validate` function, we see that there are a few requirements for accepting a new message.
The message must:
* belong to the same round
* be the first message from a particular sender and owner

These two requirements guarantee that we only accept one message for a particular round, sender, and owner.
Once we have messages for a particular round and owner from `2f+1` unique senders, we make a decision.

### Message Echo
How are messages propagated in the protocol?
When an owner initiates the broadcast protocol, the `owner` and `sender` fields will match.
This is the indication to a peer that it should `echo` the message to other peers.

In a byzantine environment, it is not guaranteed that the broadcaster is honest.
Therefore, peers must echo what they have heard from the broadcaster to other peers.
Once each process has enough information, it can make a decision.

### Decision
The trickiest part of the broadcast protocol is the decision.
According to our fault model, we _must_ make progress after receiving `2f+1` messages.
The question is, how many of those messages must contain the same value in order to accept the value?

I posit that all `2f+1` messages must contain the same value.
Suppose that a process makes a decision after receiving `2f` of the same value
