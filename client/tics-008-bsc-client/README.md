| tics  | title      | stage | category | kind          | requires |
| ----  | ---------- | ----- | -------- | ------------- | -------- |
| 8     | BSC Client | draft | TIBC/TAO | instantiation | 2        |

## Synopsis

This specification document describes a blockchain client (verification algorithm) that uses the BSC `parlia` consensus (hereinafter referred to as the `parlia` consensus).

### Motivation

State machines of various sorts replicated using the BSC `parlia` consensus algorithm might like to interface with other replicated state machines or solo machines over `TIBC`.

### Definitions

Functions & terms are as defined in [TICS 2](../../core/tics-002-client-semantics).

`currentTimestamp` is as defined in [TICS 24](../../core/tics-024-host-requirements).

### Desired Properties

This specification must satisfy the client interface defined in `TICS-002`.

## Technical Specification

This specification depends on the [`palia` consensus algorithm].

### Headers

The `BSC` client `Header`s completely adopts the definition of `Ethereum`, except that the `Extra` field is extended, and the signature information of the validators who committed the block is added.

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

In BSC, the DPOS+POA method is adopted for consensus. At present, the validator set is updated every `200` blocks. If the height of a block is a multiple of 200, then it is called an epoch block. The `header.Extra` in the epoch block saves the validator set for the next epoch, and the data format is:

```text
  extraVanity = [32]byte{}
  validateSets = N * [20]byte{}
  signature = [65]byte{}
  header.Extra = extraVanity + validateSets + signature
```

Non-epoch Blocks:

```text
  extraVanity = [32]byte{}
  signature = [65]byte{}
  header.Extra = extraVanity  + signature
```

The message signature algorithm of the header is:

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

### Client States

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

- `ChainId`: The unique identifier of a blockchain
- `Period`: The number of seconds between blocks to be compulsorily executed, which is set as 3 in the current BSC client
- `Epoch`: The number of blocks produced during each epoch, which is set as 200 in the current BSC client
- `Validators`: The validator set of the current epoch
- `RecentSingers`: The validator set of the latest signatures (equal opportunities).
- `ContractAddress`: The addresses of cross-chain management contracts
- `TrustingPeriod`: The trusting period (ns) of the current client

### Consensus State

The `BSC` client saves the timestamp, height and state root information at the current height.

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
  assert(consState.Timestamp+cs.TrustingPeriod > now()) return Expired
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

### Client initialisation

The `BSC` consensus algorithm incorporates the concept of validators, and is divided into epoch blocks and non-epoch blocks. Only epoch blocks will update the validator set, so epoch blocks must be uploaded during initialization to save the current validator set.

```go
func (cs ClientState) Initialize(clientStore sdk.KVStore,consState ConsensusState) {
    assert(cs.Header.Number % cs.Epoch != 0, "must be epoch block")
    validatorBytes := cs.Header.Extra[extraVanity : len(cs.Header.Extra)-extraSeal]
    for _, v := range ParseValidators(validatorBytes) {
      cs.Validators[v.Address] = struct{}{}
    }
}
```

### The Block Period of Delayed Acknowledgement

Any critical application of `BSC` may have to wait for `2/3*N+1` blocks to be relatively safely finalized.

```go
func (cs ClientState) DelayBlock() uint64{
  return (2*len(cs.Validators)/3 + 1)
}
```

### The Time Period of Delayed Acknowledgement

Returns the time period of delayed acknowledgement of the current light client.

```go
func (cs ClientState) DelayTime() uint64 {
  return (2*len(cs.Validators)/3 + 1) * cs.Period
}
```

### Validity predicate

The validity check of the `BSC` client is basically the same as the Ethereum light client. The difference is that the verification of the BSC block header is based on the epoch.


Except for legitimacy of the block header itself, the difficulty coefficient, change of validators, and the mechanism of equal opportunity also need to be verified when verifying block headers.

- The change of validator set occurrs in the `header.Number % cs.Epoch` block
- The change of validator set takes place in the `header.Number % cs.Epoch == uint64(len(cs.Validators)/2)` block.
- To prevent malicious nodes from continuously producing blocks, `Parlia` specifies that each verified node can only propose one block in a continuous `limit` blocks, what is to say, in each round, at most `len(cs.Validators) - limit` nodes can propose blocks, wherein `limit = floor(len(cs.Validators) / 2) + 1` .

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

### State verification functions

BSC client state verification functions check a Merkle proof against a previously validated commitment root.

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
