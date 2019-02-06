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
