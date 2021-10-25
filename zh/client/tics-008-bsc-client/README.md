| tics  | title                                | stage | category | kind   | requires |
| ----  | ------------------------------------ | ----- | -------- | ------ | -------- |
| 8     | BSC 轻客户端                          | 草案   | IBC/TAO  | 实例化  | 2        |

## 概要

本规范文档描述了使用BSC `parlia`共识的区块链客户端（验证算法），以下简称`parlia`共识

### 动机

使用BSC `parlia`共识算法复制的各种状态机可能希望通过`TIBC`与其他复制状态机或单机连接。

### 定义

功能和术语在 [TICS 2](../../core/tics-002-client-semantics) 中定义。

`currentTimestamp` 在 [TICS 24](../../core/tics-024-host-requirements) 中定义。

### 所需属性

该规范必须满足`TICS-002`中定义的客户端接口。

## 技术指标

该规范取决于[`parlia`共识算法](https://github.com/binance-chain/docs-site/blob/master/docs/smart-chain/guides/concepts/consensus.md)。

### 区块头

`BSC`客户端的`Header`完全采用`Ethereum`的定义，只是对`Extra`字段进行扩展，添加出块人的签名信息等。

```go
type Header struct {
  ParentHash  [32]byte
  UncleHash   [32]byte
  Coinbase    [20]byte
  Root        [32]byte
  TxHash      [32]byte
  ReceiptHash [32]byte
  Bloom       [256]byte
  Difficulty  *big.Int
  Number      *big.Int
  GasLimit    uint64
  GasUsed     uint64
  Time        uint64
  Extra       []byte
  MixDigest   [32]byte
  Nonce       [8]byte
}
```

在BSC中采用了DPOS+POA的方式进行共识，目前每隔`200`个区块进行一次验证人集合的更新，如果一个区块的高度为200的倍数，则称为纪元区块。纪元块中的header.Extra保存了下一个纪元的验证人集合，数据格式为：

```text
  extraVanity = [32]byte{}
  validateSets = N * [20]byte{}
  signature = [65]byte{}
  header.Extra = extraVanity + validateSets + signature
```

非纪元块：

```text
  extraVanity = [32]byte{}
  signature = [65]byte{}
  header.Extra = extraVanity  + signature
```

区块头的消息签名算法为：

```go
func encodeSigHeader(w io.Writer, header *types.Header, chainId *big.Int) {
  err := rlp.Encode(w, []interface{}{
    chainId,
    header.ParentHash,
    header.UncleHash,
    header.Coinbase,
    header.Root,
    header.TxHash,
    header.ReceiptHash,
    header.Bloom,
    header.Difficulty,
    header.Number,
    header.GasLimit,
    header.GasUsed,
    header.Time,
    header.Extra[:len(header.Extra)-65],
    header.MixDigest,
    header.Nonce,
  })
  if err != nil {
    panic("can't encode: " + err.Error())
  }
}

func SealHash(header *types.Header, chainId *big.Int) (hash common.Hash) {
  hasher := sha3.NewLegacyKeccak256()
  encodeSigHeader(hasher, header, chainId)
  hasher.Sum(hash[:0])
  return hash
}

```

### 客户端状态

```go
type ClientState struct{
  Header           Header
  ChainId          *big.Int
  Epoch            uint64
  Period           uint64
  Validators       map[[32]byte]struct{}
  RecentSingers    map[uint64][32]byte
  ContractAddress  [20]byte
  TrustingPeriod   uint64
}
```

- `ChainId`: 区块链的唯一标识符。
- `Period`：要强制执行的块之间的秒数，当前BSC设置为3
- `Epoch`：每个纪元经历的块数，当前BSC设置为200。
- `Validators`：当前纪元的验证人集合
- `RecentSingers`：最近签名的验证人集合(机会均等)
- `ContractAddress`：跨链管理合约地址
- `TrustingPeriod`：当前轻客户端的信任周期(ns)

### 共识状态

`BSC`客户端保存了当前高度的时间戳、高度以及状态根信息。

```go
type ConsensusState struct{
  Timestamp uint64
  Number    *big.Int
  Root      []byte
}
```

### 状态

```go
func (cs ClientState) Status(clientStore sdk.KVStore) {
  onsState, err := GetConsensusState(clientStore, cdc, cs.GetLatestHeight())
  assert(err == nil) return Unknown
  assert(consState.Timestamp+cs.TrustingPeriod > now()) return Expired
}
```

### 默克尔证明

```go
type Proof struct {
  Address       string
  Balance       string
  CodeHash      string
  Nonce         string
  StorageHash   string
  AccountProof  []string
  StorageProof  []StorageResult
}

// StorageProof ...
type StorageResult struct {
  Key   string
  Value string
  Proof []string
}
```

### 客户端初始化

`BSC`的共识算法中加入了验证人的概念，而且分为纪元区块和非纪元区块，只有纪元区块才会更新验证人集合，所以需要在初始化时必须上传纪元区块以保存当前的验证人集合。

```go
func (cs ClientState) Initialize(clientStore sdk.KVStore,consState ConsensusState) {
    assert(cs.Header.Number % cs.Epoch != 0, "must be epoch block")
    validatorBytes := cs.Header.Extra[extraVanity : len(cs.Header.Extra)-extraSeal]
    for _, v := range ParseValidators(validatorBytes) {
      cs.Validators[v.Address] = struct{}{}
    }
}
```

### 延迟确认区块周期

`BSC`的任何关键应用程序可能必须等待`2/3*N+1`块才能确保相对安全的终结

```go
func (cs ClientState) DelayBlock() uint64{
  return (2*len(cs.Validators)/3 + 1)
}
```

### 延迟确认时间周期

返回当前轻客户端的延迟确认时间周期。

```go
func (cs ClientState) DelayTime() uint64 {
  return (2*len(cs.Validators)/3 + 1) * cs.Period
}
```

### 有效性断言

`BSC` 客户端有效性检查大致和以太坊的轻客户端相同，区别在于：BSC的区块头的校验是基于纪元的。

验证区块头时，除了验证区块头自身的合法性以外，还需要验证难度系数、验证人变化、机会均等机制等：

- 验证人集合变更事件发生在`header.Number % cs.Epoch`块
- 验证人集合变更生效发生在`header.Number % cs.Epoch == uint64(len(cs.Validators)/2)`
- 为避免某些恶意节点持续出块，`Parlia`中规定每一个认证节点在连续`limit`个区块中，最多只能签发一个区块，也就是说，每一轮中，最多只有`len(cs.Validators) - limit`个认证节点可以参与区块签发。其中`limit = floor(len(cs.Validators) / 2) + 1`。

```go

var (
  diffInTurn = big.NewInt(2)  // Block difficulty for in-turn signatures
  diffNoTurn = big.NewInt(1)  // Block difficulty for out-of-turn signatures
)           

func (cs ClientState) CheckHeaderAndUpdateState(header Header) {
    // assert header timestamp is past timestamp
    assert(header.Time > currentTime)
    assert(length(header.Extra) >= 32+65)
    assert(header.MixDigest != []byte{})
    assert(header.Difficulty != nil)
    assert(header.GasLimit <= uint64(0x7fffffffffffffff))
    assert(header.GasUsed <= header.GasLimit)
    assert(cs.Header.Number + 1 =  header.Number)
    assert(cs.Header.Hash() = header.ParentHash)
    verifyCascadingFields(header, cs.Header)

    signer = ecrecover(header, signature, cs.ChainId)
    if signer != header.Coinbase {
      return errCoinBaseMisMatch
    }

    // The validator set change occurs at `header.Number % cs.Epoch == 0`
    if header.Number % cs.Epoch == 0 {
       validatorBytes := checkpoint.Extra[extraVanity : len(checkpoint.Extra)-extraSeal]
       validators = ParseValidators(validatorBytes)
       set("clientState/snapshot/{header.Number}",validators)
    }

    // Delete the oldest validator from the recent list to allow it signing again
    if limit := uint64(len(cs.Validators)/2 + 1); header.Number >= limit {
       delete(cs.RecentSingers, header.Number-limit)
    }

    cs.RecentSingers[header.Number] = signer

    // The validator set change takes effect on `header.Number % cs.Epoch == len(cs.Validators)/2`
    if header.Number % cs.Epoch == uint64(len(cs.Validators)/2){
      epochNumber := header.Number / cs.Epoch
      epochHeight := epochNumber * cs.Epoch
      validators := get("clientState/snapshot/{epochHeight}")

      newVals := make(map[common.Address]struct{}, len(validators))
      for _, val := range validators {
          newVals[val] = struct{}{}
      }

      oldLimit := len(clientState.Validators)/2 + 1
      newLimit := len(newVals)/2 + 1
      if newLimit < oldLimit {
        for i := 0; i < oldLimit-newLimit; i++ {
          delete(cs.RecentSingers, header.Number-uint64(newLimit)-uint64(i))
        }
      }
      clientState.Validators := newVals
    }

    if _, ok := cs.Validators[signer]; !ok {
      return errUnauthorizedValidator
    }

    for seen, recent := range cs.RecentSingers {
      if recent == signer {
        // Signer is among RecentSingers, only fail if the current block doesn't shift it out
        if limit := uint64(len(cs.Validators)/2 + 1); seen > number-limit {
          return errRecentlySigned
        }
      }
    }

    inturn := cs.inturn(signer)
    if inturn && header.Difficulty.Cmp(diffInTurn) != 0 {
      return errWrongDifficulty
    }
    if !inturn && header.Difficulty.Cmp(diffNoTurn) != 0 {
      return errWrongDifficulty
    }

    clientState.Header = header
    consensusState = ConsensusState{header.Time, header.Number, header.Root}
    return clientState,consensusState
}

func (cs ClientState) inturn(validator common.Address) bool {
  offset := cs.Header.Number % uint64(len(cs.Validators))
  return validators[offset] == validator
}
```

### 状态验证功能

BSC客户端状态验证功能根据先前验证的承诺根检查Merkle证明。

```go
func (cs ClientState) VerifyPacketCommitment(
    height Height,
    prefix Prefix,
    proof []byte,
    sourceChain string,
    destChain string,
    sequence uint64,
    commitmentBytes bytes) {

    assert(cs.DelayBlock() <= cs.Header.Number - height)
    path = applyPrefix(prefix, "commitments/{sourceChain}/{destChain}/packets/{sequence}")
    assert(sha256(path) == proof.StorageProof[0].Key)
    // verify whether it is the management contract address
    assert(proof.Address == cs.ContractAddress)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{clientState.ChainName()}/consensusStates/{height}")
    // verify that the provided commitment has been stored
    trie.VerifyProof(root, account, proof.AccountProof)
    v := trie.VerifyProof(root, storageHash, proof.StorageProof)
    assert(commitmentBytes == v)
}

func (cs ClientState) VerifyPacketAcknowledgement(
  height Height,
  prefix Prefix,
  proof []byte,
  sourceChain string,
  destChain string,
  sequence uint64,
  acknowledgement bytes) {
    assert(cs.DelayBlock() <= cs.Header.Number - height)
    path = applyPrefix(prefix, "acks/{sourceChain}/{destChain}/acknowledgements/{sequence}")
    assert(sha256(path) >= proof.StorageProof[0].Key)
    // verify whether it is the management contract address
    assert(proof.Address == cs.ContractAddress)

    // check that the client is at a sufficient height
    assert(clientState.Header.Height >= height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{clientState.ChainName()}/consensusStates/{height}")
     // verify that the provided commitment has been stored
    trie.VerifyProof(root, account, proof.AccountProof)
    v:= trie.VerifyProof(root, storageHash, proof.StorageProof)
    assert(commitmentBytes == v)
}

func (cs ClientState) VerifyPacketCleanCommitment(
  height Height,
  prefix CommitmentPrefix,
  proof []byte,
  sourceChain string,
  destChain string,
  sequence uint64) {
    assert(cs.DelayBlock() <= cs.Header.Number - height)
    path = applyPrefix(prefix, "cleans/{sourceChain}/clean")
    assert(sha256(path) >= proof.StorageProof[0].Key)
    // verify whether it is the management contract address
    assert(proof.Address == cs.ContractAddress)

    // check that the client is at a sufficient height
    assert(clientState.Header.Height >= height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{clientState.ClientID()}/consensusStates/{height}")
    // verify that the provided commitment has been stored
    trie.VerifyProof(root, account, proof.AccountProof)
    v := trie.VerifyProof(root, storageHash, proof.StorageProof)
    assert(sequence == v)
}
```
