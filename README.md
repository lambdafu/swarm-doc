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

### Synchronizing State

Synchronize state by exchanging RON frames. Each peers state is a collection of CRDT instances (objects) consisting of a RON frame (state frame) and an read-only, internal representation (e.g. C++ object).

The root of each state change is a RON frame `update`. The state frame `state` and the `update` are combined using the type depend reducer function, yielding a new state frame. The reducer is selected by inspecting the header op of the update frame. The type UUID of the header op determines the reducer. The header ops object UUID determines which state frame to use.

Each frame is processed atomically. If the type or object is unknown or the reducer fails, the frame is discarded as a whole and no change takes place.

```
header := getHeaderOp(update)
reduce := TYPES[header[0]]
new_state := reduce(OBJECTS[header[2]], update)
OBJECTS[header[2]] <- new_state
```

If the update is a single raw op, it's first converted to a frame. A artificial frame header op is generated with the same type, object and event UUIDs. The location UUID empty. The header ops start and end versions are equal to the event UUID.
