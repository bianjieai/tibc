
| tics | title            | stage | category | kind      | requires | author              | created    | modified   |
| ---- | ---------------- | ----- | -------- | --------- | -------- | ------------------- | ---------- | ---------- |
| 2    | Client Semantics | draft | TIBC/TAO | interface | 23,24    | zhiqiang@bianjie.ai | 2021-07-23 | 2021-07-26 |

## Synopsis

This specification describes the full life cycle of the light client, including create, update, upgrade and definitions of the entire process of verifying source chain data using the light client state. Implementation details of each light client is beyond the scope of this specification.

### Motivation

In the TIBC protocol, an actor, which may be an end user, an off-chain process, or a machine, needs to be able to verify the state updates which the other machine's consensus algorithm has agreed upon, and reject any possible updates which the other machine's consensus algorithm has not agreed upon. A light client is an algorithm with which a machine can do so. This standard formalizes the light client model and requirements, so that the IBC protocol can easily integrate with new machines which are running new consensus algorithms as long as associated light client algorithms fulfilling the listed requirements are provided.

### Definitions

- `get`, `set`, `Path` and `Identifier` are as defined in TICS 24.
- `CommitmentRoot` is as defined in TICS 23. It must provide an inexpensive way for downstream logic to verify whether key/value pairs are present in state at a particular height.
- `ConsensusState` is an interface type representing the state of a validity predicate. `ConsensusState` must be able to verify state updates agreed upon by the associated consensus algorithm. It must also be serializable in a canonical fashion so that third parties, such as counterparty machines, can check that a particular machine has stored a particular consensus state. It must finally be introspectable by the state machine which it is for, such that the state machine can look up its own consensus state at a past height.
- `ClientState` is an interface type representing the state of a client. A `ClientState` must expose query functions to verify membership or non-membership of key/value pairs in state at particular heights and to retrieve the current `ClientState`.

### Desired Properties

Light clients must provide a secure algorithm to verify other chains' canonical `header`s, using the existing `ConsensusState`. The higher-level abstractions will then be able to verify sub-components of the state with the `CommitmentRoot` stored in the `ConsensusState`, which are guaranteed to have been committed by the other chain's consensus algorithm. 

Validity predicates are expected to reflect the behaviour of the full nodes which are running the corresponding consensus algorithm. Given a `ConsensusState` and a list of messages, if a full node accepts the new `Header` generated with `Commit`, then the light client MUST also accept it, and if a full node rejects it, then the light client MUST also reject it. 

Light clients are not replaying the whole message transcript, so it is possible under cases of consensus misbehaviour that the light clients' behaviour differs from the full nodes'. In this case, a misbehaviour proof that proves the divergence between the validity predicate and the full node can be generated and submitted to the chain so that the chain can safely deactivate the light client, invalidate past state roots, and await higher-level intervention.

## Technical Specification

### Data Storage

- ClientState

```go
func SetClientState(chainName string, clientState ClientState) {
    store := k.ClientStore(ctx, chainName)
    store.Set("clientState", clientState)
}

func ClientStore(chainName string) sdk.KVStore{
    return "clients/{chainName}"
}
```

- ConsensusState

```go
func SetConsensusState(chainName string, height Height, consensusState ConsensusState) {
    store := k.ClientStore(ctx, chainName)
    store.Set("consensusStates/{height}", consensusState)
}
```

- ChainName

`ChainName` specifies the name of the chain, which must be specified in genesis and be written to the blockchain during initialization (`InitGenesis`) of the chain, also, `ChainName` must be unique to create its own light client on other chains.

```go
func SetChainName(chainName string) {
    store := ctx.KVStore(k.storeKey)
    store.Set("client/chainName", chainName)
}
```

### Data Structures

#### Client Type

The client type specifies the type of consensus algorithm used by the light client. In order to facilitate future expansion, it is defined as a constant as follows:

```golang
type ClientType = string
```

#### Header

A `Header` is an interface data structure defined by a client type that provides information to update a `ConsensusState`. A `Header` can be submitted to an associated client to update the stored `ConsensusState`. They likely contain a height, a proof, a commitment root, and possibly updates to the validity predicate. Different blockchains have different ways of implementation, so the block header is defined as an interface:

```golang
type Header interface {
    ClientType() string
    GetHeight() Height
    ValidateBasic() error
}
```

#### Height

`Height` defines the current block height information of the destination blockchain. Considering the possibility of forks, it should also contain some other information like revision. The definition is as follows:

```go
type Height interface {
    IsZero() bool
    LT(Height) bool
    LTE(Height) bool
    EQ(Height) bool
    GT(Height) bool
    GTE(Height) bool
    GetRevisionNumber() uint64
    GetRevisionHeight() uint64
    Increment() Height
    Decrement() (Height, bool)
    String() string
}
```

#### State

`State` defines the state of the current light client, such as `Active`, `Expired`, and `Unknown`, which can be defined as follows:

```go
    type Active  = "Active"
    type Expired = "Expired"
    type Unknown = "Unknown"
```

The specific state judgment logic is implemented by the specific light client. When the light client expires, its state cannot be updated directly, but can only be updated by an upgrade proposal.

#### ClientState

`ClientState` is an interface data structure defined by a client type. It may keep an arbitrary internal state to track verified roots and past misbehaviours. `ClientState` consists of a series of interfaces, which work jointly to check the validity of the cross-chain data packet.

- ClientType
`ClientType` is used to return the definition of the consensus algorithm type used by the current light client.

```go
func (cs ClientState) ClientType() string
```

- The unique identifier of the light client

```go
func (cs ClientState) ChainName() string
```

- The latest height

Returns the latest height of the light client.

```go
func (cs ClientState) GetLatestHeight() Height
```

- Validity check

Check the validity of the current light client data.

```go
func (cs ClientState) Validate() error
```

- Proof verification rules

- State of the light client

Returns the state of the current light client.

```go
func (cs ClientState) Status() Status
```

- The time period of delayed acknowledgement

Returns the time period of delayed acknowledgement of the current light client.

```go
func (cs ClientState) DelayTime() uint64
```

- The block period of delayed acknowledgement

Returns the block period of delayed acknowledgement of the current light client, for example, Bitcoin requires an acknowledgement period of more than 6 blocks.

```go
func (cs ClientState) DelayBlock() uint64
```

- MerklePath prefix

The MerklePath prefix of the current light client, as defined in `tics-023`.

```go
func (cs ClientState) Prefix() Prefix
```

- Initialize the light client

```go
func (cs ClientState) Initialize(consensusState consensusState) error
```

- Verify and update the light client state

```go
func (cs ClientState) CheckHeaderAndUpdateState(header Header) (ClientState, ConsensusState, error)
```

- Verify the cross-chain data packet

Verify the received cross-chain data packet using the consensus state of the light client and information such as proofs.

```go
func (cs ClientState) VerifyPacketCommitment(
    height Height,
    prefix Prefix,
    proof []byte,
    sourceChain string,
    destChain string,
    sequence uint64,
    commitmentBytes []byte,
) error
```

The specific parameters are explained as follows:

- `height`: The height of proof in the current cross-chain data packet.
- `prefix`: The name of store in the cross-chain data packet (storeKey).
- `proof`: The merkle proof of the cross-chain data packet.
- `sourceChain`: The source chain of the data packet.
- `destChain`: The destination chain of the data packet.
- `sequence`: The sequence of the cross-chain data packet.
- `commitmentBytes`: The sequence of the cross-chain data packet.

- Verify the cross-chain data packet Ack

Verify the cross-chain data packet acknowledgement using the consensus state of the light client and information such as proofs.

```go
func (cs ClientState) VerifyPacketAcknowledgement(
    height Height,
    prefix Prefix,
    proof []byte,
    sourceChain string,
    destChain string,
    sequence uint64,
    acknowledgement []byte
) error
```

The specific parameters are explained as follows:

- `height`: The height of proof in the current cross-chain data packet.
- `prefix`: The name of store in the cross-chain data packet (storeKey).
- `proof`: The merkle proof of the cross-chain data packet.
- `sourceChain`: The source chain of the data packet.
- `destChain`: The destination chain of the data packet.
- `sequence`: The sequence of the cross-chain data packet.
- `acknowledgement`: The commitment of the cross-chain data packet acknowledgement (the hash value of the cross-chain data packet acknowledgement is calculated following the same rules).

- Verify the cross-chain data packet (light client state cleanup)

If it is needed to clean up the expired state information of the light client, a cleanup light client cross-chain data packet can be sent to clean up the state.

```go
func (cs ClientState) verifyPacketCleanCommitment(
    height Height,
    prefix Prefix,
    proof []byte,
    sourceChain string,
    destChain string,
    sequence uint64,
    cleanCommitmentBytes []byte
) error
```

The specific parameters are explained as follows:

- `height`: The height of proof in the current cross-chain data packet.
- `prefix`: The name of store in the cross-chain data packet (storeKey).
- `proof`: The merkle proof of the cross-chain data packet.
- `sourceChain`: The source chain of the data packet.
- `destChain`: The destination chain of the data packet.
- `sequence`: The sequence of the cross-chain data packet.
- `cleanCommitmentBytes`: The commitment of the cleanup light client data packet (the hash value of the cross-chain data packet acknowledgement is calculated following the same rules).

- ConsensusState

`ConsensusState` defines the consensus result of the light client at a given height, which is usually defined in the form of `merkle root`. The `merkle root` here can either be storage root or transaction root, the specific implementation of which is defined by the light client.

```go
type ConsensusState interface {
    ClientType() string // Consensus kind

    // GetRoot returns the commitment root of the consensus state,
    // which is used for key-value pair verification.
    GetRoot() []byte

    // GetTimestamp returns the timestamp (in nanoseconds) of the consensus state
    GetTimestamp() uint64

    ValidateBasic() error
}
```

## Transaction

### Update the light client state

Upon submitting a cross-chain data packet, a user must update the light client state first before submitting the transaction. Under the situation where the source chain does not offer instant finality and needs to delay the acknowledgement to confirm the transaction, the user should update the light client state N blocks before submitting the cross-chain data package. For example, if the source chain needs to delay the acknowledgement for 6 blocks, then the proof height of the cross-chain data packet submitted by the user must meet the following requirement:

```text
 proofHeight - clientState.GetLatestHeight() >= 6
```

The transaction structure of updating the light client is defined as:

```go
type MsgUpdateClient struct {
    ChainName string
    Header Header
}
```

The process of updating the light client is as follows:

```go
func UpdateClient(chainName string,header Header) error {
    AssertClientExist(chainName)
    clientState := k.GetClientState(ctx, chainName)
    newClientState, newConsensusState := clientState.CheckHeaderAndUpdateState(header)
    SetClientState(chainName, newClientState)
    SetConsensusState(chainName,header.GetHeight() newConsensusState)
    return nil
}
```

## Management of the light client

The TIBC light client is managed through governance on the hub, or other approaches such as multi-signature on other blockchains.

### CreateClientProposal

```go
type CreateClientProposal struct {
    ChainName      string
    ClientState    ClientState
    ConsensusState ConsensusState
}
```

On Ethereum, `ChainName` refers to the contract address of a light client deployed on Ethereum, where `CreateClientProposal` is used instead to register and initialize the light client for the management contract. Note that the writing method of the light client can only be called by the management contract to avoid modification of the contract's state and ensure the security of the light client. This process is briefed as follows:


```text
deploy the management contract -> deploy the light client contract -> register light client for the management contract -> Update the light client contract state
```

An example:

```go
func CreateClient(chainName string,clientState ClientState,consensusState ConsensusState) (string, error){
    AssertClientNotExist(chainName)

    SetClientState(chainName, clientState)
    if err := clientState.Initialize(consensusState); err != nil {
        return "", err
    }
    
    SetConsensusState(chainName,clientState.GetLatestHeight() newConsensusState)
    return nil
}
```

### UpgradeClientProposal

When the light client expires or the state of which is incorrect, an upgrade can be proposed to force the light client state to be updated.

```go
type UpgradeClientProposal struct {
    ChainName      string
    ClientState    ClientState
    ConsensusState ConsensusState
}
```

An example of the process is as follows:

```go
func UpgradeClient(ctx sdk.Context, chainName string, upgradedClient ClientState, upgradedConsState ConsensusState){
    AssertClientExist(chainName)
    SetClientState(chainName, upgradedClient)
    SetConsensusState(chainName,updatedClientState.GetLatestHeight() newConsensusState)
}
```

### RegisterRelayerProposal

In order to ensure timely and complete relay of cross-chain data packets to the destination chain, a relayer needs to be deployed to scan the transactions on the source chain and relay them to the destination chain. The relayer needs to complete the following tasks:

- Update the state of the light client of the source chain on the destination chain.
- Relay cross-chain transactions to the destination chain.

Considering the security of cross-chain data, relayers are registered through a voting method. The relayer chain needs to authenticate the source of data and accept transactions sent by authenticated relayers only. The RegisterRelayerProposal is designed as follows:

```go
type RegisterRelayerProposal struct {
    ChainName      string
    Relayers       []string
}
```

```go
func RegisterRelayer(ctx sdk.Context, chainName string, relayers []sdk.AccAddress){
    AssertClientExist(chainName)
    SetRelayers(chainName, relayers)
}

func AuthRelayer(ctx sdk.Context, chainName string, relayer sdk.AccAddress) bool {
    AssertClientExist(chainName)
    return GetRelayer(chainName).Contains(relayer)
}
```
