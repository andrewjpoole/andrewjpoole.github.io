---
layout: post
title: Could private network Blockchain enable leaderless consensus in a distributed system? part 1
image: /images/chain.jpg
published: true
tags:
  - blockchain
  - distributed systems
  - leaderless consensus
---

Ever since I first heard a presentation about Blockchain in a DLL (developer learning lunch) at work a while ago,
I have been convinced that it has a huge number of potential uses.
A list which can be recreated identically in a number of places,
with an easy method of verification which can prove it has not been changed, is surely a very useful thing in software systems.
I am interested in whether a private network Blockchain could be used as the source of truth for shared state
in a distributed system, allowing consensus _without_ needing a leader...

## Background

Over the last two or three years, my team has been building a distributed system, a set of dotNetCore microservices using EventStore, Consul and ElasticSearch. There have been times where a certain task needed to completed exactly once even though we had 5 nodes in the cluster, so which node should perform the task? We used advice from a paper in the documentation of Consul, which is a fantastic tool made by Hashicorp. Consul has a concept of sessions and locks, where the nodes in a system can all attempt to place their own session token into the lock, one will succeed and perform the task and the others will wait for their turn.

## Whats wrong with having a leader?

Even with the help of Consul, which is designed to help with the problems involved with building distributed systems, leader election code is necessarily complex. Only one node should be leader at any one time. All of the other nodes should know who the leader is and that they themselves are not the leader. If the current leader dies, another node should take over without losing any tasks that should have been performed. We found that, for a time sensitive task, even leaders in waiting needed to listen for and attempt to perform all of the leader's tasks, just in case they became the leader.

## Super basic explanation of Blockchain

Anyway, back to Blockchain! Blockchain is a brilliant concept of a chain of blocks, where each block contains some data, the previous block's hash and the block's own hash. There is a brilliant graphical explanation [here](http://graphics.reuters.com/TECHNOLOGY-BLOCKCHAIN/010070P11GN/index.html), but I will attempt my own explanation anyway...  

First we have to explain hashes.
A hash is a unique value that can be mathematically calculated from a given piece of data using an algorithm. The hash of a given piece of data will always be the same, meaning if you send the data to another place along with the original hash, the arrived data can be hashed again and if the two hashes match, you can be sure that arrived data is identical to the original data that was sent. It is not possible to derive the data from the hash, the hash is like a digital fingerprint of the data.

The first block in the chain is special in that its previous hash is null, its called the genesis block.

Each block's hash contains the previous block's hash, or in other words the fingerprint of a block is made up of the data in that block AND the previous block's fingerprint. Its possible to loop through all of the blocks in the chain checking the hashes and proving that the chain is valid. If the chain is instantiated in two locations and both have the same blocks added, the chains (including all of the hashes) will be identical. If one chain has any differences, however small, the hashes will be different from the altered block onwards, the difference between the two chains will be obvious.

So, as long as there are a number of chains (and preferably an odd number) _and_ less than 50% of them have been tampered with, the system can detect the tampering and reject the altered blocks.

## Whats the difference between public and private network Blockchain?

In a crypto currency, there is a large public network of nodes, in fact anyone can run a node and have a copy of the chain. The source of truth is therefore completely decentralised and as long as no one party owns more than 50% of the nodes, the system can be trusted.

![Blockchain Mining rig](/images/miningrig.jpg "Picture of a Blockchain Mining rig")
Public network Blockchain also has a concept called 'proof of work' which makes it non trivial to be able to create the next block, one example is that the hash must start with a certain number of leading zeros, this is where the concept of mining comes from. A number called a nonce is added to the block and if the hash doesn't meet the constraint (e.g. starting with 11 zeros) the nonce would be incremented and the hash recalculated, this happens repeatedly until the hash meets the constraint. This would probably mean that a special piece of code running on a high-end graphics card on a 'mining rig' would have been churning out millions of hashes for about 10 minutes before the next block was found. This is a very simplified explanation, in BitCoin for instance there is also a confirmation process explained [here](https://coincenter.org/entry/how-long-does-it-take-for-a-bitcoin-transaction-to-be-confirmed)

Its this trust that makes public network blockchain potentially useful for things like currency, banking/financial stuff, supply chain provenance, asset tracking, contracts, medical records etc and loads more.

In a private network, all of the nodes are trusted so there is no need for proof-of-work and the next plain old hash is perfectly fine, which is very fast compared to finding the next hash which starts with 11 zeros.

## I feel a side project coming on...

So my idea (and I know I'm probably not the first to have had it) is that a private network Blockchain should be able to enable a distributed system to agree on the state of the truth without needing to elect a leader.

Each node in the distributed system would have its own copy of the Blockchain. Any new data added should be broadcasted to the other nodes, who will attempt to verify their Blockchain with the new block on the end. If they cant verify the chain, they will reject the new block, sending a negative response and the original node will have to try again.

So in a cluster with nodeA, nodeB and nodeC, if new data is added to nodeA and nodeB at the same time, one of them (lets say nodeB) will succeed in sending the new block to nodeC first, the other (nodeA) will (having received the new data from nodeB itself) recreate the new block and try again.

This gets more interesting as the number of nodes increases and my guess is that as long as I know the total number of nodes and therefore the size of a quorum (i.e. the number of nodes that represents more than half of the total) a given node should continue pushing a new block as long as it doesn't receive more than the quorum amount of negative responses and it should definitely continue pushing a block if it receives more than the quorum amount of positive responses! The larger the cluster the more chance of competition, this is the bit that I am most looking forward to experimenting with.

I needed a concrete use case to test my theory and happened to have 50 odd Gb of family photos which need sorting so my plan was to build a WebApi which will allow me to store and retrieve files (mostly photos) and to define and search on metadata associated with the files I have stored. The files will be stored in an Azure blob store and the metadata will be stored in a Blockchain. The WebApi should run on multiple nodes as a distributed system, where all nodes are capable of reading and writing data. Whenever data is written to a node it should broadcast the new data to the other nodes and therefore all nodes should have an identical Blockchain.

## How will I know when the question has been answered?

I guess I will have my system up and running, receiving uploaded photos as fast as possible for a sustained period of time, while being able to simultaneously make read successful requests and metadata searches. At any point in time, I should be able to check the latest block's hash matches across the cluster.

## Progress so far

So far, I have:

* developed the basic dotNetCore WebApi service with Swagger, starting using LiteDb behind an abstraction
* developed my Blockchain implementation with unit tests
* developed a P2P module using the excellent GRPC
* developed a Blockchain database and switched it in place of LiteDB.
* written some NUnit integration tests that add 3000 random strings of data to a random node in a cluster of three, as fast as possible and check that the Blockchains are identical at the end
* persisted the chain to an Azure blob after each write
* added some ridiculously fast indexing using the humble Dictionary class.
* added logging to Elasticsearch
* have centralised configuration in Hashicorp Consul

## Next steps

Next, I plan to:

* put the source onto GitHub
* write tests which prove that the cluster does not get into a split brain scenario after two coincident writes
* improve the Blockchain persistence
* have new nodes initialise their Blockchains from the blob store
* periodically check for and prune dead nodes from the list.
* develop an Angular frontend
* add some kind of decent load testing

## Future ambitions

* I need an uploader application.
* I want to add indexing, including geospatial metadata, in ElasticSearch
* I want to deploy the thing to Azure Kubernetes Service.
* Long term I'm thinking of using Cognitive services Face Api and maybe some machine learning to fill in missing Tags etc.

Thanks for reading!
