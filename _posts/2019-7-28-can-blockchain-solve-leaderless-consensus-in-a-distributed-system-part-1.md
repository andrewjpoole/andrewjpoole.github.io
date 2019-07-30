---
layout: post
title: Can Blockchain solve leaderless consensus in a distributed system part 1
published: false
tags:
  - blockchain
  - distributed systems
  - leaderless consensus
---

Ever since I first heard a presentation about Blockchain in a DLL (developer learning lunch!) at work a while ago,
I have been convinced that it has a huge number of potential uses.
A list which can be recreated identically in a number of places,
with an easy method of verification which can prove it has not been changed, is surely a very useful thing in software systems.
I am interested in whether a private network Blockchain could be used as the source of truth of shared state
in a distributed system _without_ needing to have a leader...

## Background

Over the last two or three years, my team has been building a distributed system, a set of dotNetCore microservices using EventStore, Consul and ElasticSearch. There have been times where a certain task needed to completed exactly once even though we had 5 nodes in the cluster, so which node should perform the task? We used advice from a paper in the documentation of Consul, which is a fantastic tool made by Hashicorp. Consul has a concept of sessions and locks, where the nodes in a system can all attempt to place their own session token into the lock, one will succeed and perform the task and the others will wait fir their turn.

## Whats wrong with having a leader?

complex code, some systems only the leader can write etc

## Super basic explanation of Blockchain

Anyway, back to Blockchain! Blockchain is a brilliant concept of a chain of blocks, whereby each block contains some new data, the previous block's hash and the block's own hash (which is a hash of all of the properties of the current block).

The first block in the chain is special in that its previous hash is null, its called the genesis block.

So each block's hash contains the previous block's hash, this means that if the chain is initiated in two locations and both have the same blocks added, the chains will be identical. If one chain has any differences, however small, the hashes will be different from the altered block onwards, the difference between the two chains will be obvious.

So as long as there are a number of chains (and preferably an odd number) _and_ less than 50% of them have been tampered with, the system can detect the tampering and reject the altered blocks.

include some simple code and maybe a link

## Whats the difference between public and private network Blockchain?

In a crypto currency, there is a large public network of nodes, in fact anyone can run a node, the chain is not secret. 

some stuff about proof of work/stake and nonces etc

## I feel a side project coming on...

include needing a concrete usecase and also needing to store half a Terrabyte of family photos etc.

## How will I know when the question has been answered?

## Progress so far

## Next steps



