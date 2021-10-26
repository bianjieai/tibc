| tics | title                   | stage | category | kind          | requires | author                           | created    | modified   |
| ---- | ----------------------- | ----- | -------- | ------------- | -------- | -------------------------------- | ---------- | ---------- |
| 4    | Port & Packet Semantics | draft | TIBC/TAO | instantiation | 2, 3, 26 | Wenxi Cheng <vincent@bianjie.ai> | 2021-07-23 | 2021-07-26 |

## Synopsis

`Port` specifies the port allocation system by which modules can bind to uniquely named ports allocated by the TIBC handler. Ports can then be used to pass `Packet` and can be transferred or later released by the module which originally bound to them.

`Packet` defines the standard of cross-chain data packets. The modules which send and receive TIBC packets decide how to construct packet data and how to act upon the incoming packet data, and must utilise their own application logic to determine which state to apply in transactions according to what data the packet contains.

### Motivation

The interblockchain communication protocol uses a cross-chain message passing model. IBC packets are relayed from one blockchain to the other by external relayer processes. Chain `A` and chain `B` confirm new blocks independently, and packets from one chain to the other may be delayed, censored, or re-ordered arbitrarily. Packets are visible to relayers and can be read from a blockchain by any relayer process and submitted to any other blockchain.

The TIBC protocol must provide exactly-once delivery guarantees to allow applications to reason about the combined state of connected modules on two chains. For example, an application may wish to allow a single tokenized asset to be transferred between and held on multiple blockchains while preserving fungibility and conservation of supply. The application can mint asset vouchers on chain `B` when a particular IBC packet is committed to chain `B`, and require outgoing sends of that packet on chain `A` to escrow an equal amount of the asset on chain `A` until the vouchers are later redeemed back to chain `A` with an IBC packet in the reverse direction. This ordering guarantee along with correct application logic can ensure that total supply is preserved across both chains and that any vouchers minted on chain `B` can later be redeemed back to chain `A`.

### Definitions

`ConsensusState` is as defined in [TICS 2 ](../tics-002-client-semantics).

`Port` is a particular kind of identifier that is used to issue permission to channel opening and usage to modules.

```go
enum Port {
  FT,
  NFT,
  SERVICE,
  CONTRACT,
}
```

`module` is a sub-component of the host state machine independent of the TIBC handler. Examples include Ethereum smart contracts and Cosmos SDK & Substrate modules. The TIBC specification makes no assumptions of module functionality other than the ability of the host state machine to use object-capability or source authentication to permission ports to modules.

`hash` is a generic collision-resistant hash function, the specifics of which must be agreed on by the modules utilising the channel. `hash` can be defined differently by different chains. 

 `Packet` is a particular interface by which a cross-chain data packet is defined:

```go
interface Packet {
  sequence: uint64
  port: Identifier
  sourceChain: Identifier
  destChain: Identifier
  relayChain: Identifier
  data: bytes
}
```

- The `sequence` number corresponds to the order in which the packet is sent.
- `port` identifies the ports on both sending and receiving chain.
- `sourceChain` identifies the sending chain of the data packet.
- `destChain` identifies the receiving chain of the data packet.
- `relaychain` identifies the relay chain of the data packet, which, when showing null, means that relay is not required.
- `data` is an opaque value that can be defined by the application logic of the associated modules.

> Note that `Packet` is never directly serialised. Rather it is an intermediary structure used in certain function calls that may need to be created or processed by modules calling the TIBC handler. 

> `OpaquePacket` is a data packet, but cloaked in an obscuring data type by the host state machine, such that a module cannot act upon it other than to pass it to the TIBC handler. The TIBC handler can cast a packet to an OpaquePacket and vice versa.

```go
type OpaquePacket = object
```

`CleanPacket` defines the cross-chain data packet used to clean up the state.

```go
interface CleanPacket {
  sequence: uint64
  sourceChain: Identifier
  destChain: Identifier
  relayChain: Identifier
}
```

- `sequence` identifies the maximum sequence number of the data to be cleaned up.
- `sourceChain` identifies the sending chain of the data packet.
- `destChain` identifies the receiving chain of the data packet.
- `relaychain` identifies the relay chain of the data packet, which can be null.

> Note that the `CleanPacket` execution is idempotent, thus doesn't include the sequence attribute itself to avoid repeated operations.

### Desired Properties

#### Efficiency

- The speed of packet transmission and confirmation should be limited only by the speed of the underlying chains. Proofs should be batchable where possible.

#### Exactly-once delivery

- TIBC packets sent on one end of a channel should be delivered exactly once to the other end.
- If one or both of the chains halt, packets may be delivered no more than once, and once the chains resume packets should be able to flow again.

#### Permissioning

## Technical Specification

### Dataflow visualisation

### Preliminaries

#### 存储路径

`nextSequenceSend` is an unsigned integer counter store path that identifies the sequence number for the next packet to be sent.

```go
function nextSequenceSendPath(sourceChainIdentifier: Identifier, destChainIdentifier: Identifier): Path {
  return "seqSends/{sourceChainIdentifier}/{destChainIdentifier}/nextSequenceSend"
}
```

Constant-size commitments to packet data fields are stored under the `packetCommitmentPath`:

```go
function packetCommitmentPath(sourceChainIdentifier: Identifier, destChainIdentifier Identifier, sequence: uint64): Path {
    return "commitments/{sourceChainIdentifier}/{destChainIdentifier}/packets/" + sequence
}
```

Packet receipt data are stored under the `packetReceiptPath`:

```go
function packetReceiptPath(sourceChainIdentifier: Identifier, destChainIdentifier Identifier, sequence: uint64): Path {
    return "receipts/{sourceChainIdentifier}/{destChainIdentifier}/receipts/" + sequence
}
```

Packet acknowledgement data are stored under the `packetAcknowledgementPath`:

```go
function packetAcknowledgementPath(sourceChainIdentifier: Identifier, destChainIdentifier Identifier, sequence: uint64): Path {
    return "acks/{sourceChainIdentifier}/{destChainIdentifier}/acknowledgements/" + sequence
}
```

Packet history cleanup data are stored under the `packetCleanPath`:

```go
function packetCleanPath(chainIdentifier: Identifier): Path {
    return "cleans/{chainIdentifier}/clean"
}
```

### Sub-protocols

#### Identifier validation

#### Packet flow & handling

##### Packet lifecycle

The following sequence of steps must occur for a packet to be sent from module *1* on machine `A` to module *2* on machine `B`, starting from scratch.

The module can interface with the TIBC handler through [TICS 26 ](../tics-026-routing-module).

##### Sending Packets

The sendPacket function is called by a module in order to send a TIBC packet on a channel end owned by the calling module to the corresponding module on the counterparty chain.

Calling modules MUST execute application logic atomically in conjunction with calling `sendPacket`. 

The TIBC handler performs the following steps in order:

- Checks that the client on the receiving chain is available
- Checks that the calling module owns a sending port
- Increments the send sequence counter associated with the channel
- Stores a constant-size commitment to the packet data & packet timeout

Note that the full packet is not stored in the state of the chain - merely a short hash-commitment to the data & timeout value. The packet data can be calculated from the transaction execution and possibly returned as log output which relayers can index.

```go
function sendPacket(packet: Packet) {
    nextSequenceSend = provableStore.get(nextSequenceSendPath(packet.sourceChain, packet.destChain))
    abortTransactionUnless(packet.sequence === nextSequenceSend)

    // all assertions passed, we can alter state
    nextSequenceSend = nextSequenceSend + 1
    provableStore.set(nextSequenceSendPath(packet.sourceChain, packet.destChain), nextSequenceSend)
    provableStore.set(packetCommitmentPath(packet.sourceChain, packet.destChain, packet.sequence), hash(packet.data))

    // log that a packet has been sent
    emitLogEntry("sendPacket", {sequence: packet.sequence, data: packet.data, sourceChain: packet.sourceChain, destChain: packet.destChain, relayChain: packet.relayChain})
}
```

#### Receiving packets

The `recvPacket` function is called by a module in order to receive a TIBC packet sent on the corresponding channel end on the counterparty chain. 

In conjunction with calling `recvPacket`, calling modules MUST either execute application logic or queue the packet for future execution. 

TIBC handler performs the following steps in order:

- Checks that the client on the sending chain is available
- Checks that the calling module own a reveiving port
- Checks the inclusion proof of packet data commitment in the sending chain's state
- Sets a store path to indicate that the packet has been received
- Increments the packet receive sequence associated with the channel end

```go
function recvPacket(
  packet: OpaquePacket,
  proof: CommitmentProof,
  proofHeight: Height): Packet {
    abortTransactionUnless(packet.relaychain === packet.sourceChain)
    abortTransactionUnless(packet.relaychain === packet.destChain)

    signChain = packet.sourceChain
    if !packet.relayChain.isEmpty() && selfChain == packet.destChain {
      signChain = packet.relayChain
    }
    client = provableStore.get(clientPath(signChain))

    abortTransactionUnless(client.verifyPacketData(
      proofHeight,
      proof,
      packet.sourceChain,
      packet.destChain,
      packet.sequence,
      concat(packet.data)
    ))

    // all assertions passed (except sequence check), we can alter state
    if selfChain == packet.relaychain {
      // store commitment
      provableStore.set(packetCommitmentPath(packet.sourceChain, packet.destChain, packet.sequence), hash(packet.data))

      // log that a packet has been sent
      emitLogEntry("sendPacket", {sequence: packet.sequence, data: packet.data, sourceChain: packet.sourceChain, destChain: packet.destChain, relayChain: packet.relayChain})
    }

    if selfChain == packet.destChain{
      // recive packet
      abortTransactionUnless(provableStore.get(packetReceiptPath(packet.sourceChain, packet.destChain, packet.sequence) === null))
      provableStore.set(
        packetReceiptPath(packet.sourceChain, packet.destChain, packet.sequence),
        "1"
      )

      // log that a packet has been received
      emitLogEntry("recvPacket", {sequence: packet.sequence, sourceChain: packet.sourceChain, destChain: packet.destChain, data: packet.data, relayChain: packet.relayChain})

      // return transparent packet
      return packet
    }
}
```

#### Writing Acknowledgements

The `writeAcknowledging` function is called by a module in order to write data which resulted from processing a TIBC packet that the sending chain can then verify, which is a sort of "execution receipt" or "RPC call response". 

Calling modules must execute application logic atomically in conjunction with calling `writeacknowledging`. 

This is an asynchronous acknowledgement, the contents of which do not need to be determined when the packet is received, only when processing is completed. In the synchronous case, `writeAcknowledgement` can be called in the same transaction (atomically) with `recvPacket`.

Acknowledging packets is not required; however, if an ordered channel uses acknowledgements, either all or no packets must be acknowledged (since the acknowledgements are processed in order). Note that if packets are not acknowledged, packet commitments cannot be deleted on the source chain. Future versions of TIBC may include ways for modules to specify whether or not they will be acknowledging packets in order to allow for cleanup.

`writeAcknowledging` does not check if the packet being acknowledged was actually received, because this would result in proofs being verified twice for acknowledged packets. This aspect of correctness is the responsibility of the calling module. The calling module must only call `writebackiding` with a packet previously received from recvPacket. 

TIBC handler performs the following steps in order:

- Checks that an acknowledgement for this packet has not yet been written
- Sets the opaque acknowledgement value at a store path unique to the packet

```go
function writeAcknowledgement(
  packet: Packet,
  acknowledgement: bytes): Packet {
    // cannot already have written the acknowledgement
    abortTransactionUnless(provableStore.get(packetAcknowledgementPath(packet.sourceChain, packet.destChain, packet.sequence) === null))

    // write the acknowledgement
    provableStore.set(
      packetAcknowledgementPath(packet.sourceChain, packet.destChain, packet.sequence),
      hash(acknowledgement)
    )

    // log that a packet has been acknowledged
    emitLogEntry("writeAcknowledgement", {sequence: packet.sequence, sourceChain: packet.sourceChain, destChain: packet.destChain, relayChain: packet.relayChain, data: packet.data, acknowledgement})
}
```

#### Processing acnowledgements

The `acknowledgePacket` function is called by a module to process the acknowledgement of a packet previously sent by the calling module on a channel to a counterparty module on the counterparty chain. `acknowledgePacket` also cleans up the packet commitment, which is no longer necessary since the packet has been received and acted upon. 

Calling modules may atomically execute appropriate application acknowledgement-handling logic in conjunction with calling `acknowledgePacket`.

```go
function acknowledgePacket(
  packet: OpaquePacket,
  acknowledgement: bytes,
  proof: CommitmentProof,
  proofHeight: Height): Packet {
    abortTransactionUnless(packet.relaychain === packet.sourceChain)
    abortTransactionUnless(packet.relaychain === packet.destChain)

    signChain = packet.destChain
    if !packet.relayChain.isEmpty() && selfChain == packet.sourceChain {
      signChain = packet.relayChain
    }
    client = provableStore.get(clientPath(signChain))

    // verify we sent the packet and haven't cleared it out yet
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourceChain, packet.destChain, packet.sequence)) === hash(packet.data))

    // abort transaction unless correct acknowledgement on counterparty chain
    abortTransactionUnless(client.verifyPacketAcknowledgement(
      proofHeight,
      proof,
      packet.sourceChain,
      packet.destChain,
      packet.sequence,
      acknowledgement
    ))

    if selfChain == packet.relaychain {
      // write the acknowledgement
      provableStore.set(
        packetAcknowledgementPath(packet.sourceChain, packet.destChain, packet.sequence),
        hash(acknowledgement)
      )

      // log that a packet has been acknowledged
      emitLogEntry("writeAcknowledgement", {sequence: packet.sequence, sourceChain: packet.sourceChain, destChain: packet.destChain, relayChain: packet.relayChain, data: packet.data, acknowledgement})
    }

    // delete our commitment so we can't "acknowledge" again
    provableStore.delete(packetCommitmentPath(packet.sourceChain, packet.destChain, packet.sequence))

    // return transparent packet
    return packet
}
```

##### Acknowledgement Envelope

The acknowledgement returned from the remote chain is defined as arbitrary bytes in the TIBC protocol. This data may either encode a successful execution or a failure (anything besides a timeout). There is no generic way to distinguish the two cases, which requires that any client-side packet visualiser understands every app-specific protocol in order to distinguish the case of successful or failed relay. In order to reduce this issue, we offer an additional specification for acknowledgement formats, which should be used by the app-specific protocols.

```
message Acknowledgement {
  oneof response {
    bytes result = 21;
    string error = 22;
  }
}
```

#### Cleaning up state

Packets must be acknowledged in order to be cleaned-up.

Define a new state cleanup packet to clean up the data storage generated in cross-chain packet lifecycle. The packet is able to clean up storage within itself.

##### Sending cleanup packets

The `sendCleanPacket` function is called by a module in order to send a TIBC data packet on the corresponding channel end owned by the calling module to the counterpart chain.

```go
function sendCleanPacket(packet: CleanPacket) Packet{
    provableStore.set(packetCleanPath(packet.sourceChain), packet.sequence)

    // log that a packet has been sent
    emitLogEntry("CleanPacket", {sourceChain: packet.sourceChain, destChain: packet.destChain, relayChain: packet.relayChain, sequence: packet.sequence)
    return packet
}
```

##### Receiving cleanup packets

The `recvCleanPacket` funciton is called by a module in order to receive a TIBC packet sent on the corresponding channel end on the counterparty chain.

In conjunction with calling `recvCleanPacket`, calling modules MUST either execute application logic or queue the packet for future execution.

TIBC handler performs the following steps in order:

- Checks that the client on the sending chain is available
- Sets a store path to indicate that the packet has been received
- Clean up the `Receipt` and `Acknowledgement` whose sequence number is less than the specified value.

```go
function recvCleanPacket(
  packet: CleanPacket,
  proof: CommitmentProof,
  proofHeight: Height) {
    abortTransactionUnless(packet.relaychain === packet.sourceChain)
    abortTransactionUnless(packet.relaychain === packet.destChain)

    signChain = packet.sourceChain
    if !packet.relayChain.isEmpty() && selfChain == packet.destChain {
      signChain = packet.relayChain
    }
  
    client = provableStore.get(clientPath(signChain))

    abortTransactionUnless(client.verifyCleanData(
      proofHeight,
      proof,
      packet.sourceChain,
      concat(packet.data),
    ))

    // Overwrite previous clean packet
    provableStore.set(packetCleanPath(packet.sourceChain), packet.sequence)
  
    // Clean all receipts and acknowledgements whose sequence is less than packet.sequence
    provableStore.clean(packetReceiptPath(packet.sourceChain, packet.destChain, packet.sequence))
    provableStore.clean(packetAcknowledgementPath(packet.sourceChain, packet.destChain, packet.sequence))

    // log that a packet has been received
    emitLogEntry("CleanPacket", {sequence: packet.sequence, sourceChain: packet.sourceChain, destChain: packet.destChain, relayChain: packet.relayChain)
}
```
