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

Hyperledger fabric
----
Hyperledger _Fabric_ is an exciting new work from IBM that separate the blockchain from the application.
In traditional blockchain applications, such as Bitcoin, Ethereum, etc., the blockchain and application are one monolithic unit.
Hyperledger Fabric (simply Fabric from here on), aims to solve this by segregating the application from the blockchain, and further breaking the blockchain into modular, pluggable components.
