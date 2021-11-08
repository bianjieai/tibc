| tics | title        | stage | category | kind   | requires |
| ---- | ------------ | ----- | -------- | ------ | -------- |
| 9    | ETH light client | draft  | TIBC/TAO  | instantiation | 2        |

## Synopsis

This specification document describes a client (verification algorithm) for a blockchain using EHT `PoW` consensus, which is hereinafter referred to as `PoW` consensus.

### Motivation

State machines of various sorts replicated using the Ethereum `PoW` consensus algorithm might like to interface with other replicated state machines or solo machines over `TIBC`.

### Definitions

Functions & terms are as defined in [TICS-002](../../core/tics-002-client-semantics).

### Desired Properties

This specification must satisfy the client interface defined in `TICS-002`.


## Technical Specification

This specification depends on the [PoW consensus algorithm](https://github.com/ethereum/go-ethereum/blob/master/consensus/ethash/consensus.go).

### Headers

The `Header` of the `ETH` client entirely adopts the `Ethereum` definitions.

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

### Client state

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

- `Header`: The header of the block
- `ChainId`: The unique identifier of the blockchain
- `ContractAddress`: The contract address of the cross-chain packet
- `TrustingPeriod`: The trusting period (ns) of the current light client
- `TimeDelay`: The time period of delayed acknowledgement
- `BlockDelay`: The block period of delayed acknowledgement

### Consensus State

The `ETH` client saves the timestamp, height, and state root information at the current height.

```go
type ConsensusState struct{
  Timestamp uint64
  Number    *big.Int
  Root      []byte
}
```

### State

```go
func (cs ClientState) Status(clientStore sdk.KVStore) {
  onsState, err := GetConsensusState(clientStore, cdc, cs.GetLatestHeight())
  assert(err == nil) return Unknown
  assert(consState.Timestamp+cs.GetDelayTime() <BlockTime()) return Expired
    return Active
}
```

### Merkle Proof

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

### Client initialization

The initiallization of the client starts upon creating the client. To satisfy the needs of `ETH` fork processing and `Header` verification, the index should be stored in HeaderIndex in form of  (hash+height , headerByte). To enable quick indexing of the `Header`, the key of the `Header` Index should be stored in the form of (`header.root+height` , `hash+height`).

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

### The block period of the delayed acknowledgement

The delayed acknowlegement of `ETH` should be automatically setup upon creating a light client.

```go
func (cs ClientState) DelayBlock() uint64{
  return cs.BlockDelay
}
```

### The time period of the delayed acknowledgement

The time period of returning the delayed acknowlegement of the current light client.

```go
func (cs ClientState) DelayTime() uint64 {
  return cs.TimeDelay
}
```

### Validity predicate

When verifying the block header, in addition to verifying the legitimacy of the block header itself, it is also necessary to verify the difficulty coefficient, mixed digest, fork judgment, etc.


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

Fork processing:

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

### State verification functions

`ETH` client state verification functions check a Merkle proof against a previously validated commitment root.

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