---
layout: single
title:  "Running Games on the Blockchain"
categories: 
  - coding
  - blockchain
---

The year is 2019, and one of the computer science memes that refuses to die is the blockchain.
People in my lab use "but what if we put it on the blockchain" as some sort of sick joke.
However, IBM's recent work _Hyperledger Fabric_ presents a "distributed operating system for blockchain applications."
An operating system you say?
Let's see if there is some truth to this meme by implementing Pong on the blockchain.

Hyperledger Fabric
----
Hyperledger _Fabric_ is an exciting new work from IBM that separate the blockchain from the application.
In traditional blockchain applications, such as Bitcoin, Ethereum, etc., the blockchain and application are one monolithic unit.
Hyperledger Fabric (simply Fabric from here on), aims to solve this by segregating the application from the blockchain, and further breaking the blockchain into modular, pluggable components.

Pong
----
Two paddles, one ball (square?).

## The Code
Fabric has several tutorials available at https://hyperledger-fabric.readthedocs.io/.
This project is based off of the FabCar example.
The chaincode (smart contract) and application code provide a foundation for easily slapping together a simple key-value store for the game state.

## Trouble Shooting
The first thing to make sure is that the blockchain is optimized for latency and not throughput.
This means reducing the amount of transactions per block to one to eliminate the latency of compiling a block.
This option can be set in the network's _configtx.yaml_ file under the `BlockSize` options.

After playing with these settings, I realized that the options don't seem to have an effect on the latency.
By default, and with lower (and higher) values for block size and block timeout, the transaction to update the game state (move a paddle) processes in around two seconds.
To find the source of the error, without getting into (too much) source code, I need the logs.

### Logging
The system is run in docker containers.
The output from the services (containerized apps) can be inspected with
```
docker logs -f <container>
```
The `-f` option means to "follow" the output, i.e. don't print and quit.

Initially, the logging output by the containers is not terribly useful.
In the orderer, I get no such output per transaction.
This is a problem, because the limited output from the peer (validator) indicates the latency is coming from the orderer.

![Peer Log Default](/images/games-on-the-blockchain/peer_log0.png)

We can enhance the logging output from the orderer and peer by setting the `FABRIC_LOGGING_SPEC` environment variable via the `docker-compose.yml` file.

_Two hours later..._

After fiddling around with the parameters for a while, I was getting nowhere.
The rate of transactions wasn't changing and the logging output was stuck at it's default level.
Then, I did what should have been my first step and read the `README.txt` in the `basic-network` directory.
For changes to affect the network, I had to re-`generate.sh` several components, incuding the _configuration block_ for the channel.

Once I did this, my configuration changes took effect and I was getting fast transactions (fast enough for a game)!
By simply changing the block configuration, I could achieve updates every 50ms, or 20 fps.
Granted, this is slow for a game, but for pong it may be usable.

![Peer Log New1](/images/games-on-the-blockchain/peer_log1.png)

## The Inspiration
This game was meant as a way for me to play around with hyperledger, develop an application, configure a network, and learn the architecture.
After starting this project, I also discovered [StreamChain][1] from the same team behind Hyperledger Fabric.
They make more optimizations, but the core idea is the same, _latency favored over throughput_.

## Conclusion
With the advent of popular _permission-based_ blockchains and a wider variety of deployment cases, there is a push to _centralize_ a technology that was once imagined to be the solution to a _decentralized problem_.
However, the consensus problem is not a new one; the use cases for a consensus system have a larger scope than anonymous banking (and don't incur as many costs!).
Hyperledger Fabric represents and interesting point in the design space which favors flexibility and configuration of rigitity.
Crucially, we can exploit this configurability to implement an application which achieves consensus while optimizing for latency.
In the end, I created a toy project to show how one might realize a latency-optimized application.

[1]: https://arxiv.org/pdf/1808.08406.pdf
