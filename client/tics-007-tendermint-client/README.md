| tics |      title         | stage | category |     kind      | requires |
| ---- | -----------------  | ----- | -------- | ------------- | -------- |
|  7   | Tendermint Client  | draft | TIBC/TAO | instantiation |     2    |

## Synopsis

This specification document describes a client (verification algorithm) for a blockchain using Tendermint consensus.

### Motivation

State machines of various sorts replicated using the Tendermint consensus algorithm might like to interface with other replicated state machines or solo machines over `TIBC`.

### Definitions

Functions & terms are as defined in [TICS 2](../../core/tics-002-client-semantics).

`currentTimestamp` is as defined in [TICS 24](../../core/tics-024-host-requirements).

The Tendermint light client uses the generalised Merkle proof format as defined in `ICS023`.

`hash` is a generic collision-resistant hash function, and can easily be configured. 

### Desired Properties

This specification must satisfy the client interface defined in `TICS-002` .

## Technical Specification

This specification depends on the [Tendermint consensus algorithm](https://github.com/tendermint/spec/blob/master/spec/consensus/consensus.md) and [light client algorithm](https://github.com/tendermint/spec/blob/master/spec/light-client/README.md).

### Client state

The Tendermint client state tracks the current revision, current validator set, trusting period, unbonding period, latest height, latest timestamp (block time), and a possible frozen height.

```go
type ClientState struct{
  ChainID string
  ValidatorSet List<Pair<Address, uint64>>
  TrustLevel Rational
  TrustingPeriod uint64
  UnbondingPeriod uint64
  LatestHeight Height
  LatestTimestamp uint64
  FrozenHeight Height
  MaxClockDrift uint64
  ProofSpecs: []ProofSpec
}
```

### Consensus state

The Tendermint client tracks the timestamp (block time), validator set, and commitment root for all previously verified consensus states (these can be pruned after the unbonding period has passed, but should not be pruned beforehand).

```go
type ConsensusState struct{
  Timestamp uint64
  NextValidatorsHash []byte
  Root []byte
}
```

### Height

The height of a Tendermint client consists of two `uint64`s: the revision number, and the height in the revision. 

```go
type Height struct{
  RevisionNumber uint64
  RevisionHeight uint64
}
```

Comparison between heights is implemented as follows:

```go
func Compare(a Height, b Height): Ord {
  if (a.RevisionNumber < b.RevisionNumber)
    return LT
  else if (a.RevisionNumber === b.RevisionNumber)
    if (a.RevisionHeight < b.RevisionHeight)
      return LT
    else if (a.RevisionHeight === b.RevisionHeight)
      return EQ
  return GT
}
```

This is designed to allow the height to reset to `0` while the revision number increases by `1` in order to preserve timeouts through zero-height upgrades. 

### Headers

The Tendermint client headers include the height, the timestamp, the commitment root, the complete validator set, and the signatures by the validators who committed the block.

```go
type Header struct{
  SignedHeader SignedHeader
  ValidatorSet List<Pair<Address, uint64>>
  TrustedValidators List<Pair<Address, uint64>>
  TrustedHeight Height
}
```

### Client initialisation

`Tendermint` client initialisation requires a (subjectively chosen) latest consensus state, including the full validator set. To clean up expired consensus states, the storage path of every `ConsensusState` need to be recorded for client cleanup.

```go
func (cs ClientState) Initialize(clientStore sdk.KVStore,consState ConsensusState) {
    clientStore.Set("iterateConsensusStates/{cs.LatestHeight}", "consensusStates/{cs.LatestHeight}")
    clientStore.Set("consensusStates/{cs.LatestHeight}/processTime", consState.Timestamp)
    return nil
}
```

### State

```go
func (cs ClientState) Status(clientStore sdk.KVStore) Status {
    assert(!cs.FrozenHeight.IsZero()) return Frozen

    onsState, err := GetConsensusState(clientStore, cdc, cs.GetLatestHeight())
    assert(err == nil) return Unknown
    assert(consState.Timestamp+cs.TrustingPeriod > now()) return Expired
}
```

### Validity predicate

Tendermint client validity checking uses the bisection algorithm described in the [Tendermint spec](https://github.com/tendermint/spec/tree/master/spec/consensus/light-client). 

```go
func (cs ClientState) CheckHeaderAndUpdateState(header Header) {
    // assert trusting period has not yet passed
    assert(currentTimestamp() - clientState.latestTimestamp < clientState.trustingPeriod)
    // assert header timestamp is less than trust period in the future. This should be resolved with an intermediate header.
    assert(header.timestamp - clientState.latestTimeStamp < trustingPeriod)
    // assert header timestamp is past current timestamp
    assert(header.timestamp > clientState.latestTimestamp)
    // assert header height is newer than any we know
    assert(header.height > clientState.latestHeight)
    // call the `verify` function
    assert(verify(clientState.validatorSet, clientState.latestHeight, clientState.trustingPeriod, maxClockDrift, header))
    // update validator set
    clientState.validatorSet = header.validatorSet
    // update latest height
    clientState.latestHeight = header.height
    // update latest timestamp
    clientState.latestTimestamp = header.timestamp
    // create recorded consensus state, save it
    consensusState = ConsensusState{header.timestamp, header.validatorSet, header.commitmentRoot}
    return clientState,consensusState
}
```

### State verification functions

Tendermint client state verification functions check a Merkle proof against a previously validated commitment root.

These functions utilise the `proofSpecs` with which the client was initialised. 

```go
func (cs ClientState) VerifyPacketCommitment(
    height Height,
    prefix Prefix,
    proof []byte,
    sourceChain string,
    destChain string,
    sequence uint64,
    commitmentBytes bytes) {
    path = applyPrefix(prefix, "commitments/{sourceChain}/{destChain}/packets/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check delay period has passed
    if err := verifyDelayPeriodPassed(height, cs.DelayTime(), cs.DelayBlock()); err != nil {
      return err
    }
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{clientState.ChainName()}/consensusStates/{height}")
    // verify that the provided commitment has been stored
    assert(root.verifyMembership(clientState.proofSpecs, path, hash(commitmentBytes), proof))
}

func (cs ClientState) VerifyPacketAcknowledgement(
  height Height,
  prefix Prefix,
  proof []byte,
  sourceChain string,
  destChain string,
  sequence uint64,
  acknowledgement bytes) {
    path = applyPrefix(prefix, "acks/{sourceChain}/{destChain}/acknowledgements/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{clientState.ChainName()}/consensusStates/{height}")
    // verify that the provided acknowledgement has been stored
    assert(root.verifyMembership(clientState.proofSpecs, path, hash(acknowledgement), proof))
}

func (cs ClientState) VerifyPacketCleanCommitment(
  height Height,
  prefix CommitmentPrefix,
  proof []byte,
  sourceChain string,
  destChain string,
  sequence uint64) {
    path = applyPrefix(prefix, "cleans/{sourceChain}/clean")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{clientState.ChainName()}/consensusStates/{height}")
    // verify that the nextSequenceRecv is as claimed
    assert(root.verifyMembership(clientState.proofSpecs, path, sequence, proof))
}
```
