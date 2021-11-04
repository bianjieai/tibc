| tics  | title                                | stage | category | kind   | requires |
| ----  | ------------------------------------ | ----- | -------- | ------ | -------- |
| 7     | Tendermint 客户端(Tendermint Client)  | 草案   | TIBC/TAO | 实例化  | 2        |

## 概要

本规范文档描述了使用 Tendermint 共识的区块链客户端（验证算法）。

### 动机

使用 Tendermint 共识算法复制的各种状态机可能希望通过`TIBC`与其他复制状态机或单机连接。

### 定义

功能和术语在 [TICS 2](../../core/tics-002-client-semantics) 中定义。

`currentTimestamp` 在 [TICS 24](../../core/tics-024-host-requirements) 中定义。

Tendermint 轻客户端使用`ICS 23`中定义的通用 Merkle 证明格式。

`hash` 是一个通用的抗碰撞散列函数，可以轻松配置。

### 所需属性

该规范必须满足`TICS-002`中定义的客户端接口。

## 技术指标

该规范取决于[Tendermint 共识算法](https://github.com/tendermint/spec/blob/master/spec/consensus/consensus.md) 和[轻客户端算法](https://github.com/tendermint/spec/blob/master/spec/light-client/README.md)。

### 客户端状态

Tendermint 客户端状态跟踪当前版本、当前验证器集、信任期、解绑期、最新高度、最新时间戳（区块时间）和可能的冻结高度。

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

### 共识状态

Tendermint 客户端跟踪所有先前验证的共识状态的时间戳（区块时间）、验证者集和承诺根（这些可以在解绑期过后进行修剪，但不应事先修剪）。

```go
type ConsensusState struct{
  Timestamp uint64
  NextValidatorsHash []byte
  Root []byte
}
```

### 高度

Tendermint 客户端的高度由两个 `uint64` 组成：修订号和修订中的高度。

```go
type Height struct{
  RevisionNumber uint64
  RevisionHeight uint64
}
```

高度之间的比较实现如下：

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

这旨在允许高度重置为 `0`，而修订号增加 `1`，以便通过零高度升级保留超时。

### 区块头

Tendermint 客户端标头包括高度、时间戳、提交根、完整的验证器集以及提交块的验证器的签名。

```go
type Header struct{
  SignedHeader SignedHeader
  ValidatorSet List<Pair<Address, uint64>>
  TrustedValidators List<Pair<Address, uint64>>
  TrustedHeight Height
}
```

### 客户端初始化

`Tendermint`客户端初始化需要（主观选择的）最新共识状态，包括完整的验证器集。为了清理过期的共识状态信息，需要记录每个`ConsensusState`的保存路径，方便客户端清理。

```go
func (cs ClientState) Initialize(clientStore sdk.KVStore,consState ConsensusState) {
    clientStore.Set("iterateConsensusStates/{cs.LatestHeight}", "consensusStates/{cs.LatestHeight}")
    clientStore.Set("consensusStates/{cs.LatestHeight}/processTime", consState.Timestamp)
    return nil
}
```

### 状态

```go
func (cs ClientState) Status(clientStore sdk.KVStore) Status {
    assert(!cs.FrozenHeight.IsZero()) return Frozen

    onsState, err := GetConsensusState(clientStore, cdc, cs.GetLatestHeight())
    assert(err == nil) return Unknown
    assert(consState.Timestamp+cs.TrustingPeriod > now()) return Expired
}
```

### 有效性断言

Tendermint 客户端有效性检查使用 [Tendermint 规范](https://github.com/tendermint/spec/tree/master/spec/consensus/light-client) 中描述的二分算法。

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

### 状态验证功能

Tendermint 客户端状态验证功能根据先前验证的承诺根检查 Merkle 证明。

这些函数利用初始化客户端的`proofSpecs`。

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
