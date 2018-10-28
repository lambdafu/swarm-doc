# swarm-doc

We are trying to nail down the latest version of Swarm, because we think it's a great idea.

## Table of Contents

1.  [Introduction](#introduction)
2.  [RON](#ron)
3.  [CRDT](#crdt)
4.  [SwarmDB](#swarmdb)
5.  [Applications](#applications)
6.  [Terms and Definitions](#terms-and-definitions)
7.  [FAQ](#faq)
8.  [Resources](#resources)

## Introduction

Swarm is a reactive data synchronization library and middleware by [Victor Grishchenko](https://github.com/gritzko). Swarm synchronizes your application's model  automatically, in real time.

### Overview

Swarm is based on three core concepts:

-   [Conflict-free replicated data types](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (CRDTs) to synchronize state automatically without merge conflicts.  Swarm favors operation based CRDT and guarantees [causal consistency](https://en.wikipedia.org/wiki/Causal_consistency). Swarm supports Set, LWW, Causal Set, and RGA (Replicated Growable Array).
-   [RON](https://github.com/gritzko/ron) (Replicated Object Notation), a data format for _distributed live data_.
-   Partially-Ordered Logs and the Swarm Protocol, which specifies how clients and servers talk to each other, and how state is actually synchronized across the network.

Additional contributions are:

-   A GraphQL/React component to integrate Swarm-based objects into a web application.

### Motivation

Web and mobile applications have become increasingly data heavy but rely on slow, bandwidth-constrained and unreliable wireless networks, while users have many devices and expect instant synchronization and offline capability. This means that synchronization issues arise not just with collaborative applications, but also with a single user and a single application.[🔗](http://swarmdb.net/articles/todomvc/)

On the other hand, persistent storage has become ubiquitous, which makes caching inexpensive.[🔗](https://de.slideshare.net/gritzko/swarm-34428560) The advantages of caching are: No more stalls due to data loading, possibility for offline use, and intermittent network connectivity does not cause problems. This improves the user experience. But, caching also has challenges: Data can be mutated in multiple locations and invalidation is no longer effective, so versioning and synchronization becomes necessary.[🔗](https://de.slideshare.net/gritzko/swarm-34428560)

Existing attempts are ad-hoc (Evernote) or based on [operational transforms](https://en.wikipedia.org/wiki/Operational_transformation) (Google Docs), difficult to implement and not always reliable. The incremental approach to add real-time synchronization to web applications has failed.

### Swarm Architecture

Swarm allows the development of distributed applications following the MVC architecture, while fully delegating the data caching and synchronization tasks to a dedicated layer. The application can deal with the data uniformly, no matter where it resides. To enable this, the application must be designed to use only CRDTs. We believe this is the only approach that allows to reliably support the reality of distributed data.

Like other offline-capable systems, Swarm favors availability and partition tolerance over consistency ([AP-systems](https://en.wikipedia.org/wiki/CAP_theorem)). By using [partially-ordered operation logs](https://en.wikipedia.org/wiki/Partially_ordered_set#Formal_definition) and CRDTs, Swarm achieves reliable synchronization with little implementation effort. By using the RON format, it does so efficiently.[🔗](http://swarmdb.net/articles/on-kreps/)

Many software problems can be attributed to the difficulty of mixing asynchronous communication and synchronous logic. And offline use as well as intermittent network availability are just (extreme) special cases of asynchronous communication. Swarm is built to be highly tolerant to asynchronous environments from the start.[🔗](http://swarmdb.net/articles/offline-is-async/)

### History

Swarm started in 2012 as part of the Yandex Live Letters project ([Citrea](https://github.com/gritzko/citrea-model)). Early versions were fully P2P, which creates scalability problems due to per-object logs and version vectors.[🔗](https://www.datastax.com/dev/blog/why-cassandra-doesnt-need-vector-clocks) Since v1.0, Swarm improves performance by restricting the network structure to a spanning tree with linear logs.  Please keep this in
mind when reading older documentation and code.

## RON

Replicated object notation (RON) is the language in which object states and mutations, as well as all other parts of the Swarm protocol, are expressed in Swarm. RON consists mainly of a series of UUIDs (128-bit numbers) and atoms (UUIDs, integers, floats and strings). There are many different kinds of UUIDs, depending on the context.

UUIDs provide sufficient metadata to objects and their mutations to allow the implementation of CRDTs in a network of peers.

RON features two different wire formats: _text_ and _binary_. Both offer several ways of compression, adding to the complexity. We will handle compression later, but note here that efficient compression is what makes RON and Swarm practical. Compression reduces the metadata overhead.

One particular combination of four UUIDs makes up an _operation_ (short: ops) with one UUID each for the _type_, _object_, _event_ and _location_. Several operations make up the state or mutation of an object, we call this a _frame_.

Special kinds of RON ops are used for protocol handshakes and frame headers (metadata for frames). These operations have special meaning, and often omit some of the metadata that is usually included in an operation (for example, a handshake query does not have a timestamp).

### UUIDs

UUIDs are 128 bit (16 byte), and semi-compatible with RFC4122.  One goal of RON is to allow for a human-readable, compact text representation, so RON uses Base64 encoding for 2\*60 = 2\*(10 Base64 characters) of the 128 bits.  The remaining 2\*4 bits specify the _scheme_ and _variety_ of the UUID (basically, a simple type and subtype mechanism).

    vvvv ....  .... ....  .... ....  .... ....
    .... ....  .... ....  .... ....  .... ....
    00ss ....  .... ....  .... ....  .... ....
    .... ....  .... ....  .... ....  .... ....

The text representation in RON uses special punctuation characters for the scheme (one of `$`, `%`, `+`, `-`) and variety (`x/` where `x` is a Base64 character from `0123456789ABCDEF`), and Base64 encoding for the remaining bits.  Due to compression, a zero variety (`0/`) and trailing zeros are omitted.  If the second part of a name (scheme `00` = `$`) is all zero, the punctuation for the scheme is omitted as well (`db` is equal to `db$` and `db00000000$0000000000`, but `42%` can not be compressed further).

To read the text representation of an UUID, you have to look at the middle punctuation character first (one of `$%+-`). If there is none, you can assume `$` (it's a name UUID).  Then you can look at the variety (the character before the `/`), if any (usually, the variety is `0/` and omitted).

The table below shows an example for a non-zero variety that is used to differentiate integral numbers (`0/`) and hashes (`A/`).

| Scheme  | Value | Symbol | Description            | Examples                                                   |
| ------- | ----- | ------ | ---------------------- | ---------------------------------------------------------- |
| Name    | `00`  | `$`    | Hardcoded Name         | `db`, `lww`, `mouse$joe`                                   |
| Number  | `01`  | `%`    | Number or 120-bit hash | `42%`,`3%5`(for 2-tuple `3, 5`),`A/k3R9w_2F8w%Le~6dDScsw\` |
| Event   | `10`  | `+`    | Event timestamp        | `1cDKT+joe`                                                |
| Derived | `11`  | `-`    | ?                      | `1cDKT-joe`                                                |

Note: The compression for numbers seems to be wrong, see <https://github.com/gritzko/ron/issues/35>

#### Scheme 00: Human Readable Name

##### Variety 0000: Transcendental / hardcoded name or scoped name

These UUIDs are used as global identifiers, for example `db` in the handshake protocol, for error messages such as `BadHShake`, or to indicate CRDT data types such as `lww` or `set`.

If only the first 60 bits are used, the name is global and can be up to 10 Base64 characters long.  The scheme `$` is then omitted.  Otherwise, the name is a _scoped name_ local to a client (`mouse$joe` is the `mouse` name in the `joe` replica).

##### Other varieties

The other varieties can be used for various items such as ISBN, EAN-13 bar codes, SI units, etc.

#### Scheme 01: Numbers

Decimal numbers up to 9999999999 (10 digits) can be directly represented as a UUID.  The number 42 is represented as `42%` (uncompressed: `0/42`)

A non-zero variety indicates a random or hash number with 120 bit.

#### Scheme 10: Event

Events are the powerhorse of operational CRDT.  The first half of the
UUID specifies a timestamp.  The second half the origin of the
timestamp.  Together, the UUID specifies a Lampson timestamp (local
clock).

The variety further clarifies the type of clock (Base64 calendar
`MMDHmS`, Logical, Epoch). Logical timestamps are essentially a
counter that is incremented irregardless of wall clock.  Base64
calendar time is somewhat human-readable in the RON text encoding.
And finally, epoch is the number of seconds since Jan 1st, 1970.

The origin is usually a username, or a session or replica identifier.

By using calendar time and a username, for example, these particular
Lampson timestamps also provide some additional useful information
that at the very least provides some context, for example when
inspecting the data during debugging or analyzing the protocol stream.[🔗](http://swarmdb.net/articles/lamport/)

#### Scheme 11: Derived

Same as event, but different.  FIXME: Specify difference and give practical example.

#### Special UUIDs

-   `0` zero.
-   `~` never. Time stamp UUID.
-   `~~~~~~~~~~`. Error UUID. Processing this UUIDs always fails.

### Atoms

Mutations that set a value include an atom for that value.  RON defines the following atoms: Integer, float, string, or UUID.

| Atom    | Symbol | Description                      | Examples                   |
| ------- | ------ | -------------------------------- | -------------------------- |
| UUID    | `>`    | RON UUID                         | `>mouse$joe`, `>1cDKT+joe` |
| Integer | `=`    | Integer number                   | `=42`, `=-3`, `=+1`        |
| Float   | `^`    | Floating point number            | `^3.14`                    |
| String  | `'`    | String enclosed in single quotes | `'Swarm'`                  |

### Ops

Four UUIDs and a (possibly empty) sequence of atoms together make up an operation. The four UUIDs are Type (`*`), Object (`#`), Event (`@`), Field (`:`) and Value (one of `>`, `=`, `^`, `''`).  In the text representation, the symbol is used as a prefix to the UUID.

Example: `*lww#mouse$joe@1cDKT+joe:x=99` This operation is for the object `mouse$joe` of LWW type.  It sets the value for the field `x` to `99`.  The event UUID uniquely identifies the operation, so that at any replica the merge operation can succeed idempotently.

Example: `*set#mice@1cDKT+joe>mouse$joe` This operation adds the element `mouse$joe` to the set `mice`.  The field UUID is "don't care".

#### Type

The type of an Op decides what reducer to use when merging states. Each type corresponds with a CRDT.

#### Object

The UUID of the object the op applies to.

#### Event

The UUID of the op itself. The UUID is either a time stamp, zero, never or error.

#### Location

UUID of a field or sub object. Field name for LWW objects, :w

#### Trailing Atoms

Additional arguments for the event.

### Frames

RON frames are non-empty, ordered sequences of operations, delimited by
terminators. Each op in a frame is either part of a chunk or a raw op. The last
op in a frame is terminated with `.`. Frames are processed atomically by reducers.

#### Terminators

Operations in frames also have an optional terminator.

| Terminator   | Symbol | Description                                  | Examples  |
| ------------ | ------ | -------------------------------------------- | --------- |
| Raw op       | `;`    | Raw operation                                |           |
| Query header | `?`    | Header Op.                                   |           |
| Chunk header | `!`    | Starts a chunk inside a frame. Header Op.    |           |
| Reduced op   | `,`    | Optional                                     |           |
| Frame end    | `.`    | Required for streaming transports (e.g. TCP) | `'Swarm'` |

#### Chunks

Chunks are sequences of ops inside a Frame. A chunk starts with a Header Op
that is terminated with `?` or `!` and ends before the first Op not terminated
by `,`. All Ops of a Chunk have the same object and type UUIDs.

## CRDT

To avoid merge conflicts when synchronizing states across the network,
all data types must be conflict-free replicated data types (CRDTs).
Swarm uses _operation based CRDTs_, which means that update messages
in the network do not transmit the complete state of an object, but
just the changes.  The current state of an object can be recovered by
taking the last known state, and applying all subsequent operations
using a _reducer_ function.  In Swarm, all operations are causally
ordered.

Reducer functions consume a state Frame and an update and produce a new state Frame.

    // state and update are frames
    state <- reduce(state, update)

Each reduce function has the following properties.

-   _Associative_, meaning applying a change frame as a whole or each op
    incrementally produces the same final state frame. Mathematically
    $redude(state, update) = reduce(reduce(state, update[0..n]), update[n..])$

-   _Cummutative_, meaning the order in which independent update frames are
    reduce into the state does not matter. Mathematically $reduce(reduce(state,
    update1), update2) = reduce(reduce(state, update2), update1)$

-   _Idempotent_, meaning reapplying an update Frame does not change the result.
    Mathematically $reduce(state, update) = reduce(reduce(state, update),
    update)$

The following data types are defined by Swarm.

### Last writer wins (LWW)

The last writer wins strategy resolves all merge conflicts by simply
taking the highest timestamp (or highest origin if timestamps are
identical).

Note that this design relies on synchronized clocks. Use only if the
choice of value is not important for concurrent updates occurring
within the clock skew.

-   `set()`

### Set

Sets

### Background on CRDTs

CRDTs support real-time background synchronization, versioned data,
offline work, caching, prefetching and conflict-free merge for
concurrent
changes.[🔗](https://de.slideshare.net/gritzko/swarm-34428560)

Other applications using CRDTs are:

-   [Cassandra](http://cassandra.apache.org/),
-   [Riak](http://basho.com/posts/technical/distributed-data-types-riak-2-0/).

More details can be found in the seminal paper on the topic: [A comprehensive study of Convergent and Commutative
Replicated Data Types](https://hal.inria.fr/file/index/docid/555588/filename/techreport.pdf) (Shapiro et al.).

## SwarmDB

SwarmDB implements peers in a swarm system, to which other peers and replicas can connect. The SwarmDB protocol also allows replicas to delegate to sub-replicas. To avoid version vectors, the network structure is not arbitrary. Instead, the following assumptions are made:

-   There is a spanning tree of peers, where each peer sees all changes to all objects. Each peer keeps its own local linear log of all changes as they come in (arrival order).
-   The spanning tree is extended one level to cover all replicas. Each replica subscribes to a subset of objects on its peer. The peer in turn receives all changes made by the replica.
-   The spanning tree can further be extended by replicas that fork their connection to sub-replicas, providing subscriptions to some objects known by the replica. The sub-replica in turn provides all changes made by it to the replica.

Communication is in general two-way over websockets. Each half of a connection can be in two different modes: Log-based or subscription-based. Log-based connections unconditionally receive all changes for all objects, while subscription-based connections only receive changes for subscribed objects. Again the three scenarios from above:

-   A peer-to-peer connection will have both sides of the communication channel in log-based mode, so each peer receives all changes seen by the other peer.
-   A peer-to-replica connection will use both modes: The communication from the peer to the replica will be subscription-based, so the replica only sees changes for objects it is interested in. But communication from the replica to the peer will be log-based, so the peer sees all changes made by the replica to any object.
-   A replica-to-replica connection will be similar to the peer-to-replica connection, however, it will usually be initiated by the parent replica receiving all changes from the sub-replica, so directions in the handshake are opposite of that in the two cases.

(The only case missing here is that of a communication channel with two subscription-based sides. I don't know of any application for that.)

### handshake

The initial handshake for a database `default` without authentication looks like this:

    *db #default @+ ?
    *#@ !

TODO: :ref? outbound, :ref! inbound

The same with authentication atoms:

    *db #default @+ ?
    *#@ ! auth1 auth2 auth3 ... ,

The response will include a `SEEN` timestamp, a `REPLICA` client ID, and options.

    *db #default$REPLICA @SEEN+swarm ?
    *#@ =1540652128143! // wtf?

    *#@ :optkey1 optval1,
    *#@ :optkey2 optval2,
    *#@ :optkey3 optval3,
    ...

To reconnect to a database, we pass the `SEEN` timestamp of the last seen event and the `REPLICA` client ID to the handshake:

    *db #default @SEEN+REPLICA ?
    *#@ !

#### Errors

In case of an unknown database error, the response will be:

    *db #test @~ :DBUnknown$~~~~~~~~~~ ?
    *#@: !

## Applications

### Synchronizing State

Synchronize state by exchanging RON frames. Each peers state is a collection of
CRDT instances (objects) consisting of a RON frame (state frame) and an
read-only, internal representation (e.g. C++ object).

The root of each state change is a RON frame `update`. The state frame `state`
and the `update` are combined using the type depend reducer function, yielding
a new state frame. The reducer is selected by inspecting the header op of the
update frame. The type UUID of the header op determines the reducer. The header
ops object UUID determines which state frame to use.

Each frame is processed atomically. If the type or object is unknown or the
reducer fails, the frame is discarded as a whole and no change takes place.

    header := getHeaderOp(update)
    reduce := TYPES[header[0]]
    new_state := reduce(OBJECTS[header[2]], update)
    OBJECTS[header[2]] <- new_state

If the update is a single raw op, it's first converted to a frame. A artificial
frame header op is generated with the same type, object and event UUIDs. The
location UUID empty. The header ops start and end versions are equal to the
event UUID.

## Terms and Definitions

<dl>
  <dt>RON</dt>
  <dd>Replicated Object Notation, the low-level data format for synchronization in Swarm.</dd>
  <dt>Op</dt>
  <dd>Short for Operation. A data structure in RON describing mutations on objects and other parts of the data-format.</dd>
  <dt>UUID</dt>
  <dd>Universially Unique Identification. A is a 128 bit value</dd>
  <dt>Op, Spec</dt>
  <dd>A 4-tuple of UUIDs. Type, event, object and location. (Spec is an old name for Op that sometimes occurs in source code).</dd>
  <dt>Atom</dt>
  <dd>A number, a string or a UUID.</dd>
  <dt>Frame</dt>
  <dd>A non-empty, ordered sequence of ops, each with an optional terminator. Ops inside a frame are either part of a chunk or a raw op.</dd>
  <dt>Chunk</dt>
  <dd>A non-empty, ordered sequence of terminated ops. The first op is terminated with `!` or `?`, followed by zero or more ops terminated with `,`. The chunks end before the first ops not terminated by `,`.</dd>
</dl>

## Notes

Fragments of knowledge that have not been incorporated in the above text.

-   Causal Trees for collaborative real-time editing can offer a replacement for OT (operational transforms) that is offline-first, perfectly cachable, runs in the browser (JS, contentEditable), can provide authorship attribution and change detection. For Swarm, initially developed for letters.yandex.ru.[🔗](https://de.slideshare.net/gritzko/swarm-34428560)
-   Swarm combines many technologies: synchronize in real-time (WebSocket), cache data at the client (WebStorage), load and work completely offline (Application Cache). Swarm can even synchronize multiple browser tabs locally by webstorage
    events[🔗](http://swarmdb.net/articles/todomvc/).

## FAQ

### Are CRDTs really that powerful? What does it even mean to "merge states"?

Obviously, merge can not be semantically correct in many cases. Take a
collaborative editor for an example. There is no way for the machine
to merge your ideas with the other guy's ideas to produce a valid
text. It does a technically correct merge of symbol sequences, that's
it.

Every system has its limits and the wisdom is not to bore deeper than
those limits allow.[🔗](http://swarmdb.net/articles/todomvc/#comment-2098959815)

### Does Swarm accumulate cruft in the logs?

Pruning is possible in CTs once we define "replica rot time". For
example, any replica that was offline for more than a week is
considered "rotten", so either its new ops have to be reapplied anew
(to an updated replica) or it has to be
dropped.[🔗](http://swarmdb.net/articles/todomvc/#comment-2098959815)

Log pruning/compaction only affects tombstones and overwritten
values. While the past history is garbage collected, the current state
stays intact.[🔗](http://swarmdb.net/articles/todomvc/#comment-2103303260)

## Resources

-   [Swarm Homepage](https://swarmdb.net/)

### Presentations

-   [ReactiveConf 2017 - Victor Grishchenko: RON: Replicated Object Notation](https://www.youtube.com/watch?v=0Xx9kkTMi10)
-   [Replicated Object Notation (RON): like JSON, but for data sync by Victor Grishchenko (@gritzko)](https://www.youtube.com/watch?v=fb6UnKVkwVA) (React Vienna)
-   [What do Reactive apps react to? | Victor Grishchenko | Reactive 2015](https://www.youtube.com/watch?v=LDBkoixNgKs)
-   [Slides](https://de.slideshare.net/gritzko/presentations) Gritzko on SlideShare

### Publications

-   Victor Grishchenko: [“Citrea and Swarm: partially ordered op logs in the browser”](http://www.st.ewi.tudelft.nl/victor/polo.pdf), short paper at PaPEC'14
-   V. Grishchenko [“Deep hypertext with embedded revision control implemented in regular expressions”](http://www.st.ewi.tudelft.nl/victor/articles/ctre.pdf), (ACM) WIKISYM 2010, Gdansk, Poland, [slides](http://www.st.ewi.tudelft.nl/victor/articles/wikisym.html)

### Repositories by Gritzko

-   [RON](https://github.com/gritzko/ron) Most recent RON implementation in Go.
-   [Swarm](https://github.com/gritzko/swarm) Swarm client in Javascript.
-   [RON Tests](https://github.com/gritzko/ron-test) Test cases for RON.
-   [RON C++](https://github.com/gritzko/swarmcpp) C++ implementation of RON.
-   [RON JS](https://github.com/gritzko/monorepko) JS implementation of RON.
-   [RON Java](https://github.com/gritzko/monorepko) Older Java implementation of RON.
-   [Swarm.js Blog](https://github.com/gritzko/swarmjs.github.io) Older blog articles about Swarm.
-   [Swarm-RON-Docs](https://github.com/gritzko/swarm-ron-docs) Older documentation for previous versions of RON/Swarm.
-   [Rocksdb](https://github.com/gritzko/rocksdb) Fork of Rocksdb with patches for swarmdb.
-   [TODO MVC](https://github.com/gritzko/todomvc-swarm) Example Swarm+React project

### Repositories by Olebedev

-   [Swarm](https://github.com/olebedev/swarm) Olebedev's fork of the JS Swarm client.
-   [CRDT Research](https://github.com/olebedev/research-CRDT) Olebedev CRDT Research
-   [todo app](https://github.com/olebedev/todo), [chat app](https://github.com/olebedev/chat), [mice](https://github.com/olebedev/mice) Example apps for swarmdb
-   [docker-rocksdb](https://github.com/olebedev/docker-rocksdb)
