| ics | title                                      | stage  | category                        | kind       | requires  | required-by     | author                               | created  | modified  |
| ---------------- | ---------------------------------------------- | ---------- | ----------------------------------- | -------------- | ------------- | --------------------- | ---------------------------------------- | ------------ | ------------- |
| 24               | Host Environments | draft  | TIBC/TAO | interface  | 23            |  | | 2021-07-21   | 2021-07-21   |

## Synopsis

This specification defines the minimal set of interfaces which must be provided and properties which must be fulfilled by a state machine hosting an implementation of the interblockchain communication protocol.

### Motivation

TIBC is designed to be a common standard which will be hosted by a variety of blockchains & state machines and must clearly define the requirements of the host.

### Definitions

### Desired Properties

TIBC should require as simple an interface from the underlying state machine as possible to maximize the ease of correct implementation.

## Technical Specification

### Module system

The host state machine must support a module system, whereby self-contained, potentially mutually distrusted packages of code can safely execute on the same ledger, control how and when they allow other modules to communicate with them, and be identified and manipulated by a "master module" or execution environment.

The TIBC/TAO specifications define the implementations of two modules: the core "TIBC handler" module and the "TIBC relayer" module. TIBC/APP specifications further define other modules for particular packet handling application logic. TIBC requires that the "master module" or execution environment can be used to grant other modules on the host state machine access to the TIBC handler module and/or the TIBC routing module, but otherwise does not impose requirements on the functionality or communication abilities of any other modules which may be co-located on the state machine.

### Paths, identifiers, separators

An `Identifier` is a bytestring used as a key for an object stored in state, such as a connection, channel, or light client. 

Identifiers must be non-empty (of positive integer length).

Identifiers must consist of characters in one of the following categories only:
- Alphanumeric
- `. `, `_ `, `+ `, `- `, `#`
- `[ `, `] `, `< `, `>`

A `Path` is a bytestring used as the key for an object stored in state. Paths must contain only identifiers, constant strings, and the separator `"/"`.

Identifiers are not intended to be valuable resources — to prevent name squatting, minimum length requirements or pseudorandom generation may be implemented, but particular restrictions are not imposed by this specification.

The separator `"/"` is used to separate and concatenate two identifiers or an identifier and a constant bytestring. Identifiers must not contain the `"/"` character, which prevents ambiguity.

Variable interpolation, denoted by curly braces, is used throughout this specification as shorthand to define path formats, e.g. `client/{clientIdentifier}/consensusState`.

All identifiers, and all strings listed in this specification, must be encoded as ASCII unless otherwise specified.

By default, identifiers have the following minimum and maximum lengths in characters:

| Port identifier | Client identifier |
| -------------------------- | ---------------------------- |
| 2 - 64             | 9 - 64                  |

### Key/value Store

The host state machine must provide a key/value store interface with three functions that behave in the standard way:

```
type get = (path: Path) => Value | void
type set = (path: Path, value: Value) => void
type delete = (path: Path) => void
```

`Path` is as defined as above. `Value` is an arbitrary bytestring encoding of a particular data structure. Encoding details are left to separate ics.

These functions must be permissioned to the IBC handler module (the implementation of which is described in separate standards) only, so only the IBC handler module can `set` or `delete` the paths that can be read by `get`. This can possibly be implemented as a sub-store (prefixed key-space) of a larger key/value store used by the entire state machine. 

Host state machines must provide two instances of this interface - a `provableStore` for storage read by (i.e. proven to) other chains, and a `privateStore` for storage local to the host, upon which `get` , `set`, and `delete` can be called, e.g. `provableStore.set('some/path', 'value')`. 

`provableStore`:

- must write to a key/value store whose data can be externally proved with a vector commitment as defined in ICS 23
- must use canonical data structure encodings provided in these specifications as proto3 files

`privateStore`:

- may support external proofs, but is not required to - the TIBC handler will never write data to it which needs to be proved
- may use canonical proto3 data structures, but is not required to - it can use whatever format is preferred by the application environment

> Note: any key/value store interface which provides these methods & properties is sufficient for TIBC. Host state machines may implement "proxy stores" with path & value mappings which do not directly match the path & value pairs set and retrieved through the store interface — paths could be grouped into buckets & values stored in pages which could be proved in a single commitment, path-spaces could be remapped non-contiguously in some bijective manner, etc — as long as `get`, `set`, and `delete` behave as expected and other machines can verify commitment proofs of path & value pairs (or their absence) in the provable store. If applicable, the store must expose this mapping externally so that clients (including relayers) can determine the store layout & how to construct proofs. Clients of a machine using such a proxy store must also understand the mapping, so it will require either a new client type or a parameterised client.

> Note: this interface does not necessitate any particular storage backend or backend data layout. State machines may elect to use a storage backend configured in accordance with their needs, as long as the store on top fulfils the specified interface and provides commitment proofs.

### Path-space

At present, TIBC/TAO recommends the following path prefixes for the `provableStore` and `privateStore`.

Future paths may be used in future versions of the protocol, so the entire key-space in the provable store must be reserved for the TIBC handler.

Keys used in the provable store may safely vary on a per-client-type basis as long as there exists a bipartite mapping between the key formats defined herein and the ones actually used in the machine's implementation.

Parts of the private store may safely be used for other purposes as long as the TIBC handler has exclusive access to the specific keys required. Keys used in the private store may safely vary as long as there exists a bipartite mapping between the key formats defined herein and the ones actually used in the private store implementation.

Note that the client-related paths listed below reflect the Tendermint client as defined in ICS 7 and may vary for other client types.

| Store          | Path format                                                                    | Value type        | Defined in |
| -------------- | ------------------------------------------------------------------------------ | ----------------- | ---------------------- |
| provableStore  | "clients/{identifier}/clientType"                                              | ClientType        | TICS 2 |
| privateStore   | "clients/{identifier}/clientState"                                             | ClientState       | TICS 2 |
| provableStore  | "clients/{identifier}/consensusStates/{height}"                                | ConsensusState    | ICS 7  |
| privateStore   | "ports/{identifier}"                                                           | CapabilityKey     | ICS 5  |



### Module layout

Represented spatially, the layout of modules & their included specifications on a host state machine looks like so (Aardvark, Betazoid, and Cephalopod are arbitrary modules):

```
+----------------------------------------------------------------------------------+
|                                                                                  |
| Host State Machine                                                               |
|                                                                                  |
| +-------------------+       +--------------------+      +----------------------+ |
| | Module Aardvark   | <-->  | TIBC Routing Module|      | IBC Handler Module   | |
| +-------------------+       |                    |      |                      | |
|                             | Implements TICS 26 |      | Implements TICS 2,   | |
|                             |                    |      | ICS 5 internally.    | |
| +-------------------+       |                    |      |                      | |
| | Module Betazoid   | <-->  |                    | -->  | Exposes interface    | |
| +-------------------+       |                    |      | defined in ICS 25.   | |
|                             |                    |      |                      | |
| +-------------------+       |                    |      |                      | |
| | Module Cephalopod | <-->  |                    |      |                      | |
| +-------------------+       +--------------------+      +----------------------+ |
|                                                                                  |
+----------------------------------------------------------------------------------+
```

### Consensus state introspection

Host state machines must provide the ability to introspect their current height using `getCurrentHeight`: 

```
type getCurrentHeight = () => Height
```

Host state machines must define a unique `consensusstate` type fulfilling the requirements of ICS 2, with a canonical binary serialisation. 

Host state machines must provide the ability to introspect their own consensus state using `getConsensusState`: 

```
type getConsensusState = (height: Height) => ConsensusState
```

`getConsensusState` must return the consensus state for at least some number `n` of contiguous recent heights, where `n` is constant for the host state machine. Heights older than `n` may be safely pruned (causing future calls to fail for those heights). 

Host state machines must provide the ability to introspect this stored recent consensus state count `n`,using `getStoredRecentConsensusStateCount`: 

```
type getStoredRecentConsensusStateCount = () => Height
```

### Commitment path introspection

Host chains must provide the ability to inspect their commitment path using `getCommitmentPrefix`: 

```
type getCommitmentPrefix = () => CommitmentPrefix
```

The result `CommitmentPrefix` is the prefix used by the host state machine's key-value store. With the `CommitmentRoot` root and `CommitmentState` state of the host state machine, the following property must be preserved: 

```
if provableStore.get(path) === value {
  prefixedPath = applyPrefix(getCommitmentPrefix(), path)
  if value !== nil {
    proof = createMembershipProof(state, prefixedPath, value)
    assert(verifyMembership(root, proof, prefixedPath, value))
  } else {
    proof = createNonMembershipProof(state, prefixedPath)
    assert(verifyNonMembership(root, proof, prefixedPath))
  }
}
```

For a host state machine, the return value of `getCommitmentPrefix` must be constant. 

### Timestamp access

Host chains must provide a current Unix timestamp, accessible with `currentTimestamp`: 

```
type currentTimestamp = () => uint64
```

In order for timestamps to be used safely in timeouts, timestamps in subsequent headers must be non-decreasing.

### Port system

Host state machines must implement a port system, where the TIBC handler can allow different modules in the host state machine to bind to uniquely named ports. Ports are identified by an `Identifier`. 

Host state machines must implement permission interaction with the TIBC handler such that:

- Once a module is bound to a port, no other modules can use that port until the module releases it
- A single module can bind to multiple ports
- Ports are allocated first-come first-serve and "reserved" ports for known modules can be bound when the state machine is first started

This permissioning can be implemented with unique references (object capabilities) for each port (a la the Cosmos SDK), with source authentication (Ethereum), or with some other method of access control, in any case enforced by the host state machine. See ICS 5 for details.

### Datagram submission

Host state machines which implement the routing module may define a `submitDatagram` function to submit datagrams, which will be included in transactions, directly to the routing module (defined in ICS 26):

```
type submitDatagram = (datagram: Datagram) => void
```

`submitDatagram` allows relayer processes to submit TIBC datagrams directly to the routing module on the host state machine. Host state machines may require that the relayer process submitting the datagram has an account to pay transaction fees, signs over the datagram in a larger transaction structure, etc — `submitDatagram` must define & construct any such packaging required. 

### Exception system

Host state machines must support an exception system, whereby a transaction can abort execution and revert any previously made state changes (including state changes in other modules happening within the same transaction), excluding gas consumed & fee payments as appropriate, and a system invariant violation can halt the state machine.

This exception system must be exposed through two functions: `abortTransactionUnless` and `abortSystemUnless`, where the former reverts the transaction and the latter halts the state machine. 

This exception system must be exposed through two functions: `abortTransactionUnless` and `abortSystemUnless`, where the former reverts the transaction and the latter halts the state machine.

```
type abortTransactionUnless = (bool) => void
```

If the boolean passed to `abortTransactionUnless` is true, the host state machine need not do anything. If the boolean passed to `abortTransactionUnless` is false, the host state machine must abort the transaction and revert any previously made state changes, excluding gas consumed & fee payments as appropriate.

```
type abortSystemUnless = (bool) => void
```

If the boolean passed to `abortSystemUnless` is true, the host state machine need not do anything. If the boolean passed to `abortSystemUnless` is false, the host state machine must halt. 
s
### Data availability

For deliver-or-timeout safety, host state machines must have eventual data availability, such that any key/value pairs in the state can be eventually searched by relayers. For consideration of safety, data availability is not required.

For liveness of packet relay, host state machines must have bounded transactional liveness (and thus necessarily consensus liveness), such that incoming transactions are confirmed within a block height bound (in particular, less than the timeouts assign to the packets).

IBC packet data, and other data which is not directly stored in the state vector but is relied upon by relayers, must be available to & efficiently computable by relayer processes.

Light clients of particular consensus algorithms may have different and/or more strict data availability requirements.

### Event logging system

The host state machine must provide an event logging system whereby arbitrary data can be logged in the course of transaction execution which can be stored, indexed, and later queried by processes executing the state machine. These event logs are utilized by relayers to read TIBC packet data & timeouts, which are not stored directly in the chain state (as this storage is presumed to be expensive) but are instead committed to with a succinct cryptographic commitment (only the commitment is stored).

This system is expected to have at minimum one function for emitting log entries and one function for querying past logs, approximately as follows.

The function `emitLogEntry` can be called by the state machine during transaction execution to write a log entry: 

```
type emitLogEntry = (topic: string, data: []byte) => void
```

The function `queryByTopic` can be called by an external process (such as a relayer) to search all log entries associated with a given topic written by transactions which were executed at a given height. 

```
type queryByTopic = (height: Height, topic: string) => []byte[]
```

More complex query functionalities may also be supported, and may allow for more efficient relayer process queries, but is not required.

### Handling upgrades

Host machines may safely upgrade parts of their state machine without disruption to TIBC functionality. In order to do this safely, the TIBC handler logic must remain compliant with the specification, and all TIBC-related states (in both the provable & private stores) must be persisted across the upgrade. If clients exist for an upgrading chain on other chains, and the upgrade will change the light client validation algorithm, these clients must be informed prior to the upgrade so that they can safely switch atomically and preserve continuity of connections & channels.

## Backwards Compatibility

Not applicable.

## Forwards Compatibility


## Example Implementation


## Other Implementations


## History


## Copyright
