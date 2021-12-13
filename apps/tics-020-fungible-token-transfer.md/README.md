| tics  | title      | stage | category | kind          | requires |
| ----  | ---------- | ----- | -------- | ------------- | -------- |
| 020   | Fungible Token Transfer | draft | TIBC/TAO | instantiation | 004, 026        |

## Synopsis

This standard document specifies packet data structure, state machine handling logic, and encoding details for the transfer of fungible token over a TIBC port between two modules on separate chains.

## Motivation

Users of a set of chains connected over the TIBC protocol might wish to utilise an asset issued on one chain on another chain, perhaps to make use of additional features such as exchange or privacy protection, while retaining fungibility with the original asset on the issuing chain. This application-layer standard describes a protocol for transferring fungible tokens between chains connected with TIBC which preserves asset fungibility, preserves asset ownership, limits the impact of Byzantine faults, and requires no additional permissioning.

## Definitions

The TIBC handler interface & TIBC routing module interface are as defined in TICS 04 and TICS 26, respectively.

## Desired Properties

- Preservation of fungibility (two-way peg).
- Preservation of total supply (constant or inflationary on a single source chain & module).

## Technical Specification

### Data Structures

Only one packet data type is required: NonFungibleTokenPacketData, which specifies class, id, uri, sending chain, receiving chain, source chain, destination chain, and whether the FT has departed from the source chain.

```go
interface FungibleTokenPacketData {
    denomination: string
    amount: uint256
    sender: string
    receiver: string
}
```

As tokens are sent across chains using the `TICS-020` protocol, it begins to record the transfers. This information is encoded into the `denomination` field.

`denomination` field is implemented in the form of `{prefix}/{sourceChain}/{denom}` , wherein `prefix = "tibc/ft"`. The sending chain is the source chain of FT when there's no `prefix` and `sourceChain` in the field; if the field contains a `prefix` and a `sourceChain`, then the FT is transferred from the `sourceChain`. As in the case of `denomination` is `tibc/ft/A/B/denomination`, if the FT now exists on Chain `C`, then the FT is transferred from Chain `A` to Chain `B` and then to Chain `C`. If Chain `C` wishes to return the FT to its original chain, it must return the FT to Chain `B` first to update it to `tibc/ft/A`, and then return it to Chain `A` via Chain `B`.

The `awayFromOrigin` is specified when transferred from the source chain of the FT to the destination chain or returned from the destination chain to the source chain. The `awayFromOrigin` field of the packet constructed in the hops is the `awayFromOrigin` field data in the received packet.

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

The sub-protocols described herein should be implemented in an "fungible token transfer bridge" module with access to the FT transfer module and the packet module.

#### Port Setup

The setup function must be called exactly once when the module is created (perhaps when the blockchain itself is initialized) to bind to the appropriate port and create an escrow address (owned by the module). 

```go
function setup() {
    routingModule.bindPort("ftTransfer", ModuleCallbacks{
        onRecvPacket,
        onAcknowledgePacket,
        onCleanPacket,  
    })
}
```

Once the setup function has been called, cross-chain data can be transmitted through the TIBC routing module between instances of the FT transfer module on separate chains. 

#### Routing module callbacks

between chain A and Chain B:

- When the FT on Chain `A` is departing from the source chain, the FT transfer module escrows the existing FT asset on Chain `A` and mints FT on Chain `B`.
- When the FT on Chain `B` is approaching Chain `A`, the ft transfer module burns local vouchers on Chain `B` and unescrows the FT asset on Chain `A`.
- Acknowledgment data is used to handle failures, such as invalid destination accounts. Returning an acknowledgment of failure is preferable to aborting the transaction since it enables the sending chain to take appropriate action based on the nature of the failure.

`createOutgoingPacket` must be called by a transaction handler in the module which performs appropriate signature checks, specific to the account owner on the host state machine. 

```go
function createOutgoingPacket(
    denomination: string,
    amount: uint256,
    sender: string,
    receiver: string,
    awayFromOrigin: boolean,
    destChain:string,  
    relayChain:string,  
    {
        if awayFromSource {
            // escrow source FT (assumed to fail if balance insufficient)
            ft.lock(sender, denomination, amount)
        } else {
            // receiver is source chain, burn FT
            ft.burn(sender, denomination, amoun)
        }
        FungibleTokenPacketData data = FungibleTokenPacketData{denomination, amount, sender, receiver}
        handler.sendPacket(Packet{sequence, port, relayChain, destChain, data})
    }
```

`onRecvPacket` is called by the routing module when a packet addressed to this module has been received. 

```go
function onRecvPacket(packet: Packet) {
    NonFungibleTokenPacketData data = packet.data
    // construct default acknowledgement of success
    FungibleTokenPacketAcknowledgement ack = FungibleTokenPacketAcknowledgement{true, null}
    if awayFromSource { 
        if string.hasPrefix(packet.denomination){
           	// tibc/ft/A/denomination -> tibc/ft/A/B/denomination
            newDenomination = packet.sourceChain + packet.denomination 
        } else{
            // denomination -> tibc/ft/A/denomination
            newDenomination = prefix + packet.sourceChain + packet.class
        }
        ft.mint(data.receiver, newDenomination, data.amount)
    } else {
        if packet.denomination.split("/").length > 4{
            // tibc/ft/A/B/denomination -> tibc/ft/A/denomination
            newDenomination = packet.denomination - packet.destChain
        }else{
            // tibc/ft/A/denomination -> denomination
            newDenomination = packet.denomination - packet.destChain - prefix
        }
        ft.unlock(data.receiver, newDenomination, data.amount)
    }
}
```

`onAcknowledgePacket` is called by the routing module when a packet sent by this module has been acknowledged. 

```go
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // if the transfer failed, refund the FT
  if !ack.success {
    refundTokens(packet)
  }
}
```

`refundTokens` is called by `onAcknowledgePacket` to refund FT to the original sender. 

```go
function refundTokens(packet: Packet) {
  NonFungibleTokenPacketData data = packet.data
  if awayFromSource {
    // sender was source chain, unescrow tokens back to sender
    ft.unlock(data.receiver, data.denomination, data.amount)
  } else {
    // receiver was source chain, mint vouchers back to sender
    ft.mint(data.sender, data.denomination, data.amount)
  }
}
```

## Example of cross-chain single-hop

### Chain `A` transfers FT to Chain `B`

> To transfer FT from Chain `A` to Chain `B`, the relayer chain should be specified as null. Chain `A` only needs to escrow the FT, and Chain `B` creates the FT correspondingly.

Chain `A` needs to escrow the FT, then constructs a cross-chain data packet and sends it to Chain `B`

```go
FTPacket:
    denomination: iris
    amount: 100
    sender: a...
    receiver: b...
    awayFromOrigin: true
    sourceChain: A
    relayChain : ""
    destChain: B
```

Chain `B` will `mint` a new token, **newDenomination = prefix + "/" + sourceChain + "/" + denomination = tibc/ft/A/iris**

### Chain B returns the FT to Chain A

> To return the FT from Chain `B` to Chain `A`, Chain `B` burns the FT, and Chain `A` unescrows the FT correspondingly

Chain `B` needs to burn the FT, and constructs a cross-chain packet

```go
FTPacket:
    denomination: tibc/ft/A/iris
    amount: 100
    sender: b...
    receiver: a...
    awayFromOrigin: false
    sourceChain: B
    destChain: A
    relayChain: ""
```

Chain `A` will unescrow `iris`

## Example of cross-chain two-hop

### Chain A transfers the FT to Chain C through Chain B

> First Chain `A` escrows the FT under the class field, and mints an FT on Chain `B` with the class field `tibc/ft/A/iris`, then Chain `B` escrows the FT and creates a new packet which specifies Chain `C` as the destination chain, then Chain `C` will mint the FT correspondingly

Chain `A` needs to escrow the FT, and constructs a cross-chain data packet and sends it to Chain `B`

```
FTPacket:
    denomination: iris
    amount: 100
    sender: a...
    receiver: b...
    awayFromOrigin: true
    sourceChain: A
    relayChain :""
    destChain: B
```

Chain `B` will mint a new FT, with new `denomination` = prefix + "/" + sourceChain + "/" + `denomination` = `tibc/ft/A/iris`, then escrow the new FT, and reconstruct a new packet:

```go
FTPacket:
    denomination: tibc/ft/A/iris
    amount: 100
    sender: b...
    receiver: c...
    awayFromOrigin: true
    sourceChain: B
    relayChain :""
    destChain: C
```

Chain `C` will `mint` a new FT, and if a prefix already exists, then new `denomination` = `packet.class` + `packet.sourceChain` = `tibc/ft/A/B/iris`

### Chain `C` returns the FT to Chain `A` through Chain `B`

> To return the FT on Chain `C` to Chain `A`, Chain `C` will determine whether it is the source chain of the FT, if it is not, then Chain `C` burns the FT and constructs a packet, wherein `class`=`tibc/ft/A/B/iris` (the complete path from `A` to `C`), then Chain `B` unescrows and burns the FT (`class`=`tibc/ft/A/iris`), and constructs a packet to inform Chain `A` to unescrow the FT correspondingly to finish the process.

Chain `C` determines whether the FT has departed from the source chain, then burns the FT and constructs a packet and sends it to Chain `B`

```go
FTPacket:
    denomination: tibc/ft/A/B/iris
    amount: 100
    sender: c...
    receiver: b...
    awayFromOrigin: false
    sourceChain: C
    destChain: B
    relayChain: ""
```

Chain `B` receives the packet sent by Chain `C` and removes `packet.descChain` according to the packet, then creates a new `class` and performs a query on itself, if there is any result, Chain `B` unescrows and burns the FT, constructs a new packet, and sends it to Chain `A`


```go
FTPacket:
    denomination: tibc/ft/A/iris
    amount: 100
    sender: b...
    receiver: a...
    awayFromOrigin: false
    sourceChain: B
    destChain: A
    realy_chain: ""
```

Chain `A` receives the packet, determines whether the FT has departed from the source chain, then removes the prefix and checks if the FT exists on this chain, if it exists, Chain `A` will unescrow it

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
