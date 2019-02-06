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

The tools
======

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

#### Logging
The system is run in docker containers.
The output from the services (containerized apps) can be inspected with
```
docker logs -f <container>
```
The `-f` option means to "follow" the output, i.e. don't print and quit.

Initially, the logging output by the containers is not terribly useful.
In the orderer, I get no such output per transaction.
This is a problem, because the limited output from the peer (validator) indicates the latency is coming from the orderer.

![Peer Log Default](images/games-on-the-blockchain/peer_log0.png)

We can enhance the logging output from the orderer and peer by setting the `FABRIC_LOGGING_SPEC` environment variable via the `docker-compose.yml` file.

_Two hours later..._

After fiddling around with the parameters for a while, I was getting nowhere.
The rate of transactions wasn't changing and the logging output was stuck at it's default level.
Then, I did what should have been my first step and read the `README.txt` in the `basic-network` directory.

