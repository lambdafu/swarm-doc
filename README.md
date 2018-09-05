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
6. [Terms and Definitions](#terms-and-definitions)

### Existing Material

#### Repositories by Gritzko

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

Replicated object notation (RON) is the language in which object
states and mutations, as well as all other parts of the Swarm
protocol, are expressed in Swarm.  RON consists (solely!) of a series
of UUIDs (128-bit numbers), but the order of UUIDs matter, and there
are many different kinds of UUIDs, depending on the context.

UUIDs provide sufficient metadata to object and their mutations to
allow the implementation of CRDTs in a network of peers.

RON features two different wire formats: _text_ and _binary_.  Both
offer several ways of compression, adding to the complexity.  We will
handle compression later, but note here that efficient compression is
what makes RON and Swarm practical.  Compression reduces the metadata
overhead.

One particular combination of four UUIDs makes up an _operation_
(short: ops) with one UUID each for the _type_, _object_, _event_ and
_value_.  Several operations make up the state or mutation of an
object, we call this a _frame_.

Special kinds of RON ops are used for protocol handshakes and frame
headers (metadata for frames).  These degenerate operations have
special meaning, and often omit some of the metadata that is usually
included in an operation (for example, a handshake query does not have
a timestamp).


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

Events are the powerhorse of operational CRDT.  The first half of the UUID specifies a timestamp.  The second half the origin of the timestamp.  Together, the UUID specifies a Lampson-timestamp (local clock).

The variety further clarifies the type of clock (Base64 calendar `MMDHmS`, Logical, Epoch).

#### Scheme 11: Derived

Same as event, but different.  FIXME: Specify difference and give practical example.

#### Special UUIDs

- `0` zero.
- `~` never. Time stamp UUID.
- `~~~~~~~~~~`. Error UUID. Processing this UUIDs always fails.

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

#### Trailing Atoms

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

More details can be found in the seminal paper on the topic: [A comprehensive study of Convergent and Commutative
Replicated Data Types](https://hal.inria.fr/file/index/docid/555588/filename/techreport.pdf) (Shapiro et al.).

Reducer functions consume a state Frame and an update and produce a new state Frame.

```
// state and update are frames
state <- reduce(state, update)
```

Each reduce function has the following properties.

- *Associative*, meaning applying a change frame as a whole or each op
  incrementally produces the same final state frame. Mathematically
  $redude(state, update) = reduce(reduce(state, update[0..n]), update[n..])$

- *Cummutative*, meaning the order in which independent update frames are
  reduce into the state does not matter. Mathematically $reduce(reduce(state,
  update1), update2) = reduce(reduce(state, update2), update1)$

- *Idempotent*, meaning reapplying an update Frame does not change the result.
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

* `set()`

### Set

Sets

## SwarmDB

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

```
header := getHeaderOp(update)
reduce := TYPES[header[0]]
new_state := reduce(OBJECTS[header[2]], update)
OBJECTS[header[2]] <- new_state
```

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
  <dt>Spec</dt>
  <dd>A 4-tuple of UUIDs. Type, event, object and location.</dd>
  <dt>Atom</dt>
  <dd>A number, a string or a UUID.</dd>
  <dt>Frame</dt>
  <dd>A non-empty, ordered sequence of ops, each with an optional terminator. Ops inside a frame are either part of a chunk or a raw op.</dd>
  <dt>Chunk</dt>
  <dd>A non-empty, ordered sequence of terminated ops. The first op is terminated with `!` or `?`, followed by zero or more ops terminated with `,`. The chunks end before the first ops not terminated by `,`.</dd>
</dl>
