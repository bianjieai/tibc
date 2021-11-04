| tics | title        | stage | category | kind   | requires |
| ---- | ------------ | ----- | -------- | ------ | -------- |
| 9    | ETH 轻客户端 | 草案  | IBC/TAO  | 实例化 | 2        |

## 概要

本规范文档描述了使用 ETH `pow`共识的区块链客户端（验证算法），以下简称`pow`共识。

### 动机

使用 ETH`pow`共识算法复制的各种状态机可能希望通过`TIBC`与其他复制状态机或单机连接。

### 定义

功能和术语在 [TICS 2](../../core/tics-002-client-semantics) 中定义。

### 所需属性

该规范必须满足`TICS-002`中定义的客户端接口。

## 技术指标

该规范取决于[`pow`共识算法](https://github.com/ethereum/go-ethereum/blob/master/consensus/ethash/consensus.go)。

### 区块头

`ETH`客户端的`Header`完全采用`Ethereum`的定义.

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
  BaseFee     *big.Int 
}
```



### 客户端状态

```go
type ClientState struct{
  Header           	Header
  ChainId          	*big.Int
  ContractAddress  	[20]byte
  TrustingPeriod   	uint64
  TimeDelay					uint64
  BlockDelay      	uint64
}
```

-  `Header` : 区块头
- `ChainId`: 区块链的唯一标识符。
- `ContractAddress`：跨链 packet 合约地址
- `TrustingPeriod`：当前轻客户端的信任周期(ns)
- `TimeDelay`延迟确认时间周期
- `BlockDelay`延迟确认区块周期

### 共识状态

`ETH`客户端保存了当前高度的时间戳、高度以及状态根信息。

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
  assert(consState.Timestamp+cs.GetDelayTime() <BlockTime()) return Expired
    return Active
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

客户端初始化是在客户端创建的时候进行 . 为满足`ETH`的分叉处理与 Header 验证需要，在此处应以 ( hash+height , headerByte ) 结构存储到`HeaderIndex`中 . 为满足 Header 快速索引需要  , 以（ header.root+height , hash+height ）存储 Header Index 的索引 key 。

```go
func (cs ClientState) Initialize(clientStore sdk.KVStore,consState ConsensusState) {
 	header := m.Header
	headerBytes, err := cdc.MarshalInterface(&header)
	assert(err == nil) return err
	SetEthHeaderIndex(store, header, headerBytes)
	SetEthConsensusRoot(store, header.Height.RevisionHeight, header.ToEthHeader().Root, header.Hash())
	return nil
}

func EthHeaderIndexPath(hash common.Hash, height uint64) string {
	return fmt.Sprintf("%s/%s%d", KeyIndexEthHeaderPrefix, hash, height)
}
func EthHeaderIndexKey(hash common.Hash, height uint64) []byte {
	return []byte(EthHeaderIndexPath(hash, height))
}
func EthRootMainPath(root common.Hash, height uint64) string {
	return fmt.Sprintf("%s/%s%d", KeyMainRootPrefix, root, height)
}
func EthRootMainKey(root common.Hash, height uint64) []byte {
	return []byte(EthRootMainPath(root, height))
}
```

### 延迟确认区块周期

`ETH`的延迟确认应在创建轻客户端的时候自设置

```go
func (cs ClientState) DelayBlock() uint64{
  return cs.BlockDelay
}
```

### 延迟确认时间周期

返回当前轻客户端的延迟确认时间周期。

```go
func (cs ClientState) DelayTime() uint64 {
  return cs.TimeDelay
}
```

### 有效性断言

验证区块头时，除了验证区块头自身的合法性以外，还需要验证难度系数、混合摘要、分叉判断等

```go
func (cs ClientState) CheckHeaderAndUpdateState(store sdk.KVStore,currentHeader Header,header Header) {
 		// assert header not exit in HeaderIndex
  	assert(header.IsHeaderExist == false)

    // assert header timestamp is past timestamp
    parent := store.get(ConsensusStateIndexKey(header.ToEthHeader().ParentHash))
    assert(header.Time > currentTime)
    assert(length(header.Extra) >= 32)
    assert(header.MixDigest != []byte{})
    assert(header.Difficulty != nil)
    assert(header.GasLimit <= uint64(0x7fffffffffffffff))
    assert(header.GasUsed <= header.GasLimit)
    assert(parent.Hash() = header.ParentHash)
  
     // verify header 1559
    assert( VerifyEip1559Header(parentHeader, &header)!= false)
	  //verify difficulty
    expected := makedifficult(big.NewInt(9700000)(header.Time, &parent.Header)
		// 9700000 should be changed to the desired
    return verifyCascadingFields(header)
}
```
分叉处理：
```go
func (m ClientState) RestrictChain(cdc codec.BinaryCodec, store sdk.KVStore, new Header) error {
	si, ti := m.Header.Height, new.Height
	var err error
	current := m.Header
	// si > ti
	if si.RevisionHeight > ti.RevisionHeight {
    assert(store.Get(host.ConsensusStateKey(ti)) != nil)
    
		cdc.UnmarshalInterface(ConsensusTmp, &tiConsensus)
    tmpConsensus, ok := tiConsensus.(*ConsensusState)
    
		root := tmpConsensus.Root
		currentBytes := store.Get(headerIndexKey)
    assert(currentBytes != nil)
		cdc.UnmarshalInterface(currentBytes, &currentHeaderInterface)
    current = currentHeaderInterface.(*EthHeader)
		si = ti
	}
  
  // use newHashes to cache header hash
	newHashes := make([]common.Hash, 0)

	//  ti > si
	for ti.RevisionHeight > si.RevisionHeight {
		newHashes = append(newHashes, new.Hash())
		newTmp := GetParentHeaderFromIndex(store, new)
    assert(newTmp != nil)
		
    cdc.UnmarshalInterface(newTmp, &currently)
		tmpConsensus:= currently.(*Header)
		new = *tmpConsensus
    // till ti = si
		ti.RevisionHeight--
	}
  
	// when si.parent != ti.parent run to si.parent = ti.parent
	for !bytes.Equal(current.ParentHash, new.ParentHash) {
		newHashes = append(newHashes, new.Hash())
		newTmp := GetParentHeaderFromIndex(store, new)
    assert(newTmp != nil)
	 	cdc.UnmarshalInterface(newTmp, &currently);
		tmpConsensus, ok := currently.(*Header)
		new = *tmpConsensus
		ti.RevisionHeight--
		si.RevisionHeight--
		currentTmp := GetParentHeaderFromIndex(store, current)
		cdc.UnmarshalInterface(currentTmp, &currently)
		tmpConsensus = currently.(*Header)
		current = *tmpConsensus
	}
  // make newHashs cache to main_chain
	for i := len(newHashes) - 1; i >= 0; i-- {
    // get from HeaderIndex
		newTmp := store.Get(EthHeaderIndexKey(newHashes[i], ti.GetRevisionHeight()))
    cdc.UnmarshalInterface(newTmp, &currently)
		tmpHeader, ok := currently.(*Header)
		
		consensusState := &ConsensusState{
			Timestamp: tmpHeader.Time,
			Number:    tmpHeader.Height,
			Root:      tmpHeader.Root[:],
		}
		consensusStateBytes, err := cdc.MarshalInterface(consensusState)
		
		// set to main_chain
		store.Set(host.ConsensusStateKey(ti), consensusStateBytes)
		ti.RevisionHeight++
	}
	return err
}
```

### 状态验证功能

`ETH`客户端状态验证功能根据先前验证的承诺根检查Merkle证明。

```go
func (cs ClientState) VerifyPacketCommitment(
    height Height,
    prefix Prefix,
    proof []byte,
    sourceChain string,
    destChain string,
    sequence uint64,
    commitmentBytes bytes) {
  
	 ethProof, consensusState, err := produceVerificationArgs(store, cdc, m, height, proof)
		// check delay period has passed
    assert(cs.DelayBlock() <= cs.Header.Number - height)
    
  	// verify that the provided commitment has been stored
		constructor := NewProofKeyConstructor(sourceChain, destChain, sequence)
		trie.VerifyProof(ethProof,consensusState,contractAddr,commitment,proofKey)
}

func (cs ClientState) VerifyPacketAcknowledgement(
  height Height,
  prefix Prefix,
  proof []byte,
  sourceChain string,
  destChain string,
  sequence uint64,
  acknowledgement bytes) {
    ethProof, consensusState, err := produceVerificationArgs(store, cdc, m, height, proof)
		// check delay period has passed
    assert(cs.DelayBlock() <= cs.Header.Number - height)
    
  	// verify that the provided commitment has been stored
		constructor := NewProofKeyConstructor(sourceChain, destChain, sequence)
		trie.VerifyProof(ethProof,consensusState,contractAddr,commitment,proofKey)
}

func (cs ClientState) VerifyPacketCleanCommitment(
  height Height,
  prefix CommitmentPrefix,
  proof []byte,
  sourceChain string,
  destChain string,
  sequence uint64) {
    ethProof, consensusState, err := produceVerificationArgs(store, cdc, m, height, proof)
		// check delay period has passed
    assert(cs.DelayBlock() <= cs.Header.Number - height)
    
  	// verify that the provided commitment has been stored
		constructor := NewProofKeyConstructor(sourceChain, destChain, sequence)
		trie.VerifyProof(ethProof,consensusState,contractAddr,commitment,proofKey)
}
```