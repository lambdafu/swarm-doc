# swarm-doc
Newest revelations from the Gritzko Research Center! :D

We are trying to nail down the latest version of Swarm, because we
think it's a great idea.

## Table of Contents

1. [Introduction](#introduction)
2. [RON](#ron)
3. [CRDT](#crdt)
4. [SwarmDB](#swarmdb)
5. [Applications](#applications)

## Existing Material

### Repositories by Gritzko

* [RON](https://github.com/gritzko/ron) Most recent RON implementation in Go.
* [Swarm](https://github.com/gritzko/swarm) Swarm client in Javascript.
* [RON Tests](https://github.com/gritzko/ron-test) Test cases for RON.
* [RON C++](https://github.com/gritzko/swarmcpp) C++ implementation of RON.
* [RON JS](https://github.com/gritzko/monorepko) JS implementation of RON.
* [RON Java](https://github.com/gritzko/monorepko) Older Java implementation of RON.
* [Swarm.js Blog](https://github.com/gritzko/swarmjs.github.io) Older blog articles about Swarm.
* [Swarm-RON-Docs](https://github.com/gritzko/swarm-ron-docs) Older documentation for previous versions of RON/Swarm.
* [Rocksdb](https://github.com/gritzko/rocksdb) Fork of Rocksdb with patches for swarmdb.
* [TODO MVC](https://github.com/gritzko/todomvc-swarm) Example Swarm+React project

### Repositories by Olebedev

* [Swarm](https://github.com/olebedev/swarm) Olebedev's fork of the JS Swarm client.
* [CRDT Research](https://github.com/olebedev/research-CRDT) Olebedev CRDT Research
* [todo app](https://github.com/olebedev/todo), [chat app](https://github.com/olebedev/chat), [mice](https://github.com/olebedev/mice) Example apps for swarmdb
* [docker-rocksdb](https://github.com/olebedev/docker-rocksdb)

## Introduction

Swarm is a protocol for distributed data synchronization by [Victor Grishchenko](https://github.com/gritzko).  It is based on three core concepts:

* [Conflict-free replicated data types](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (CRDTs) to synchronize state automatically without merge conflicts.  Swarm favors operation based CRDT and guarantees [causal consistency](https://en.wikipedia.org/wiki/Causal_consistency).
* [RON](https://github.com/gritzko/ron) (Replicated Object Notation), a serialization format for distributed data.  Swarm supports Set, LWW, Causal Set, and RGA (Replicated Growable Array).
* Swarm Protocol, which specifies how clients and servers talk to each
  other, and how state is actually synchronized across the network.

Additional contributions are:

* A GraphQL/React component to integrate Swarm-based objects into a web application.

### History

Swarm started in 2012 as part of the Yandex Live Letters project.
Early versions were fully P2P, which creates scalability problems due
to per-object logs and version vectors.  Since v1.0, Swarm improves
performance by restricting the network structure to a spanning tree
with linear logs.  Please keep this in mind when reading older
documentation and code.

## RON

## CRDT

To avoid merge conflicts when synchronizing states across the network,
all data types must be conflict-free replicated data types (CRDTs).
Swarm uses _operation based CRDTs_, which means that update messages
in the network do not transmit the complete state of an object, but
just the changes.  The current state of an object can be recovered by
taking the last known state, and applying all subsequent operations
using a _reducer_ function.  In Swarm, all operations are causally
ordered.

More details can be found in the seminal paper on the topic: [A comprehensive study of Convergent and Commutative
Replicated Data Types](https://hal.inria.fr/file/index/docid/555588/filename/techreport.pdf) (Shapiro et al.).

The following data types are defined by Swarm.

### Last writer wins (LWW)

The last writer wins strategy resolves all merge conflicts by simply
taking the highest timestamp (or highest origin if timestamps are
identical).

Note that this design relies on synchronized clocks. Use only if the
choice of value is not important for concurrent updates occurring
within the clock skew.

* `set()`

### Set

Sets

## SwarmDB

## Applications
