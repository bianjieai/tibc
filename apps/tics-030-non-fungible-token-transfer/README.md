## Synopsis

This standard document specifies packet data structure, state machine handling logic, and encoding details for the transfer of NFT over a TIBC channel between two modules on separate chains.

## Motivation

Users of a set of chains connected over the IBC protocol might wish to utilise an asset issued on one chain on another chain, perhaps to make use of additional features such as exchange or privacy protection, while retaining non-fungibility with the original asset on the issuing chain. This application-layer standard describes a protocol for transferring non-fungible tokens between chains connected with IBC which preserves asset non-fungibility, preserves asset ownership, limits the impact of Byzantine faults, and requires no additional permissioning.

## Definitions

The TIBC handler interface & TIBC routing module interface are as defined in ICS 25 and TICS 26, respectively.

## Desired Properties

- Preservation of non-fungibility
- Preservation of total supply

## Technical Specification

### Data Structures

Only one packet data type is required: NonFungibleTokenPacketData, which specifies class, id, uri, sending chain, receiving chain, source chain, destination chain, and whether the NFT has departed from the source chain.

```go
interface NonFungibleTokenPacketData {
    class: string  // nft class
    id: string
    uri:  string  // must have value
    sender: string
    receiver: string
    awayFromOrigin:bool  // has departed from the source chain or not
}
```

As `nft` is cross-chain transferred using the `TICS-030` protocol, it begins to record the transfers. This information is encoded into the `class` field.

`class` field is implemented in the form of `{prefix}/{sourceChain}/class}` , wherein `prefix = "tibc/nft"`. The sending chain is the source chain of NFT when there's no `prefix` and `sourceChain` in the field; if the field contains a `prefix` and a `sourceChain`, then the NFT is transferred from the `sourceChain`. As in the case of `class` is `tibc/nft/A/B/nftClass`, if the NFT now exists on Chain `C`, then the NFT is transferred from Chain `A` to Chain `B` and then to Chain `C`. If Chain `C` wishes to return the NFT to its original chain, it must return the NFT to Chain `B` first to update it to `tibc/nft/A`, and then return it to Chain `A` via Chain `B`.

The `awayFromOrigin` is specified when transferred from the source chain of the NFT to the destination chain or returned from the destination chain to the source chain. The `awayFromOrigin` field of the packet constructed in the hops is the `awayFromOrigin` field data in the received packet.

The acknowledgment data type describes whether the transfer succeeded or failed, and the reason for failure (if any).

```go
type NonFungibleTokenPacketAcknowledgement = NonFungibleTokenPacketSuccess | NonFungibleTokenPacketError;

interface NonFungibleTokenPacketSuccess {
  success: "AQ=="
}

interface NonFungibleTokenPacketError {
  error: string
}
```

### Sub-protocols

The sub-protocols described herein should be implemented in an "NFT transfer bridge" module with access to the NFT module and the packet module.

#### Port Setup

The setup function must be called exactly once when the module is created (perhaps when the blockchain itself is initialized) to bind to the appropriate port and create an escrow address (owned by the module). 

```go
function setup() {
    routingModule.bindPort("nftTransfer", ModuleCallbacks{
        onRecvPacket,
        onAcknowledgePacket,
        onCleanPacket,  
    })
}
```

Once the setup function has been called, channels can be created through the TIBC routing module between instances of the nft transfer module on separate chains. 

#### Routing module callbacks

between chain A and Chain B:

- When the NFT on Chain `A` is departing from the source chain, the nft transfer module escrows the existing NFT asset on Chain `A` and mints NFT on Chain `B`.
- When NFT on Chain `B` is approaching Chain `A`, the nft transfer module burns local vouchers on Chain `B` and unescrows the NFT asset on Chain `A`.
- Acknowledgment data is used to handle failures, such as invalid destination accounts. Returning an acknowledgment of failure is preferable to aborting the transaction since it enables the sending chain to take appropriate action based on the nature of the failure.

`createOutgoingPacket` must be called by a transaction handler in the module which performs appropriate signature checks, specific to the account owner on the host state machine. 

```go
function createOutgoingPacket(
    class:string
    id: string
    uri:  string
    sender: string,
    receiver: string,
    awayFromOrigin: boolean,
    destChain:string,  
    relayChain:string,  
    {
        if awayFromSource {
            // escrow source NFT (assumed to fail if balance insufficient)
            nft.lock(sender, escrowAccount, class, id, uri)
        } else {
            // // receiver is source chain, burn NFT
            nft.BurnNFT(sender, class, id)
        }
        NonFungibleTokenPacketData data = NonFungibleTokenPacketData{class, id, uri, awayFromOrigin, sender, receiver}
        handler.sendPacket(Packet{sequence, port, relayChain, destChain})
    }
```

`onRecvPacket` is called by the routing module when a packet addressed to this module has been received. 

```go
function onRecvPacket(packet: Packet) {
    NonFungibleTokenPacketData data = packet.data
    // construct default acknowledgement of success
    NonFungibleTokenPacketAcknowledgement ack = NonFungibleTokenPacketAcknowledgement{true, null}
    if awayFromSource { 
        if string.hasPrefix(packet.class){
           	// tibc/nft/A/class -> tibc/nft/A/B/class
            newClass = packet.sourceChain + packet.class 
        } else{
            // class -> tibc/nft/A/class
            newClass = prefix + packet.sourceChain + packet.class
        }
        nft.MintNft(newClass, id, nftName, uri, nftData, sender)
    } else {
        if packet.class.split("/").length > 4{
            // tibc/nft/A/B/class -> tibc/nft/A/class
            newClass = packet.class - packet.destChain
        }else{
            // tibc/nft/A/class -> class
            newClass = packet.class - packet.destChain - prefix
        }
        nft.unlock(escrowAccount, data.receiver, newClass, id)
    }
}
```

`onAcknowledgePacket` is called by the routing module when a packet sent by this module has been acknowledged. 

```go
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // if the transfer failed, refund the NFT
  if (!ack.success)
    refundTokens(packet)
}
```

`refundTokens` is called by `onAcknowledgePacket` to refund NFT to the original sender. 

```go
function refundTokens(packet: Packet) {
  NonFungibleTokenPacketData data = packet.data
  if awayFromSource {
    // sender was source chain, unescrow NFT back to sender
    nft.unlock(escrowAccount, data.receiver, class,id)
  } else {
    // receiver was source chain, mint NFT back to sender
    nft.MintNft(class, id, nftName, uri, nftData, sender)
  }
}
```

## Example of cross-chain single-hop

### Chain `A` transfers NFT to Chain `B`

> To transfer NFT from Chain `A` to Chain `B`, the relayer chain should be specified as null. Chain `A` only needs to escrow the NFT, and Chain `B` creates the NFT correspondingly.

Chain `A` needs to escrow the NFT, then constructs a cross-chain data packet and sends it to Chain `B`

```go
NFTPacket:
    class: hellokitty
    id: id
    uri: uri
    sender: a...
    receiver: c...
    awayFromOrigin: true
    sourceChain: A
    relayChain : ""
    destChain: B
```

Chain `B` will `mint` a new NFT, **newClass = prefix + "/" + sourceChain + "/" + class = tibc/nft/A/hellokitty**

### Chain B returns the NFT to Chain A

> To return the NFT from Chain `B` to Chain `A`, Chain `B` burns the NFT, and Chain `A`  unescrows the NFT correspondingly

Chain `B` needs to burn the NFT, and constructs a cross-chain packet

```go
NFTPacket:
    class: tibc/nft/A/hellokitty
    id: id
    sender: b...
    receiver: a...
    awayFromOrigin: false
    sourceChain: B
    destChain: A
    relayChain: ""
```

Chain `A` will unescrow `hellokitty`

## Example of cross-chain two-hop

### Chain A transfers the NFT to Chain C through Chain B

> First Chain `A` escrows the NFT under the class field, and mints an NFT on Chain `B` with the class field `tibc/nft/A/hellokitty`, then Chain `B` escrows the NFT and creates a new packet which specifies Chain `C` as the destination chain, then Chain `C` will mint the NFT correspondingly

Chain `A` needs to escrow the NFT, and constructs a cross-chain data packet and sends it to Chain `B`

```
NFTPacket:
    class: hellokitty
    id: id
    uri: uri
    sender: a...
    receiver: b...
    awayFromOrigin: true
    sourceChain: A
    relayChain :""
    destChain: B
```

Chain `B` will mint a new NFT, with new `class` = prefix + "/" + sourceChain + "/" + `class` = `tibc/nft/A/hellokitty`, then escrow the new NFT, and reconstruct a new packet:


```go
NFTPacket:
    class: tibc/nft/A/hellokitty
    id: id
    uri: uri
    sender: b...
    receiver: c...
    awayFromOrigin: true
    sourceChain: B
    relayChain :""
    destChain: C
```

Chain `C` will `mint` a new NFT, and if a prefix already exists, then new `class` = `packet.class` + `packet.sourceChain` = `tibc/nft/A/B/hellokitty`

### Chain `C` returns the NFT to Chain `A` through Chain `B`

> To return the NFT on Chain `C` to Chain `A`, Chain `C` will determine whether it is the source chain of the NFT, if it is not, then Chain `C` burns the NFT and constructs a packet, wherein `class`=`tibc/nft/A/B/hellokitty` (the complete path from `A` to `C`), then Chain `B` unescrows and burns the NFT (`class`=`tibc/nft/A/hellokitty`), and constructs a packet to inform Chain `A` to unescrow the NFT correspondingly to finish the process.

Chain `C` determines whether the NFT has departed from the source chain, then burns the NFT and constructs a packet and sends it to Chain `B`

```go
NFTPacket:
    class: tibc/nft/A/B/hellokitty
    id: id
    sender: c...
    receiver: b...
    awayFromOrigin: false
    sourceChain: C
    destChain: B
    relayChain: ""
```

Chain `B` receives the packet sent by Chain `C` and removes `packet.descChain` according to the packet, then creates a new `class` and performs a query on itself, if there is any result, Chain `B` unescrows and burns the NFT, constructs a new packet, and sends it to Chain `A`


```go
NFTPacket:
    class: tibc/nft/A/hellokitty
    id: id
    sender: b...
    receiver: a...
    awayFromOrigin: false
    sourceChain: B
    destChain: A
    realy_chain: ""
```

Chain `A` receives the packet, determines whether the NFT has departed from the source chain, then removes the prefix and checks if the NFT exists on this chain, if it exists, Chain `A` will unescrow it

## Backwards Compatibility

Not applicable

## Forwards Compatibility

## Example Implementation

Coming soon.

## Other Implementations

Coming soon.

## History

## Copyright

All content herein is licensed under Apache 2.0.
