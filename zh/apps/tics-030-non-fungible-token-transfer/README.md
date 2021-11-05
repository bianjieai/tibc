## 概要

本标准文档定义了包的数据结构，状态机处理逻辑以及编码细节在两个模块之间通过TIBC通道传输NFT的详细信息。

## 动机

通过 TIBC 协议连接的一组链的用户可能希望在一条链上使用在另一条链上发行的资产，也许是为了利用额外的功能，例如交换或隐私保护，同时保留与发行链上原始资产的唯一性。 该应用层标准描述了一种用于在与 TIBC 连接的链之间传输NFT的协议，该协议保留了资产的唯一性，保留了资产所有权，限制了拜占庭故障的影响，并且不需要额外的许可。

## 定义

TIBC Handler接口和TIBC路由模块接口分别在ICS 25和TICS 26中定义。

## 要求的属性

- 保持唯一性
- 保持总供应量

## 技术规范

### 数据结构

只需要一个数据包数据类型：NonFungibleTokenPacketData，它指定Class, id，uri, 发送者,接收者,源链，目标链以及是否远离源链。

```go
interface NonFungibleTokenPacketData {
    class: string  // nft class
    id: string
    uri:  string  // 必须有值
    sender: string
    receiver: string
    awayFromOrigin:bool  // 是否远离源链 
}
```

当`nft`使用`tics-30` 协议跨链发送时，它们开始累积已传输的记录。 该信息被编码到`class`字段中。

`class`字段以`{prefix}/{sourceChain}/class}`形式实现, `prefix = "tibc/nft"`。当没有`prefix`和`sourceChain`的时候，表示发送链就是`NFT`的源头；如果带有`prefix`和`sourceChain`，则表示是从`sourceChain`转过来的。比如`tibc/nft/A/B/nftClass`，假如 C 链上存在此`NFT`，则表示`nftClass`是从 A 转到 B 再转到 C 的。如果 C 想退回此`NFT`，那么必须先退回到B，变成`tibc/nft/A`，再由 B 链退回到 A。

同时`awayFromOrigin` 在`NFT`的源头发向目标链和目标链退回到源链指定的，中间的跳转构造的包的`awayFromOrigin`字段都是获取接收包中的`awayFromOrigin`字段数据。



确认数据类型描述传输是成功还是失败，以及失败的原因（如果有）。

```go
type NonFungibleTokenPacketAcknowledgement = NonFungibleTokenPacketSuccess | NonFungibleTokenPacketError;

interface NonFungibleTokenPacketSuccess {
  success: "AQ=="
}

interface NonFungibleTokenPacketError {
  error: string
}
```

### 子协议

此处描述的子协议应在“NFT传输桥”模块中实现，该模块可访问NFT模块和 Packet模块。

#### 端口设置

必须在创建模块时（可能在初始化区块链本身时）调用 setup 函数一次以绑定到适当的端口并创建托管地址（由模块拥有）。

```go
function setup() {
    routingModule.bindPort("nftTransfer", ModuleCallbacks{
        onRecvPacket,
        onAcknowledgePacket,
        onCleanPacket,  
    })
}
```

一旦调用了 setup 函数，就可以通过 TIBC 路由模块在不同链上的nftTransfer模块的实例之间创建通道。

#### 路由模块回调

在A链和B链之间：

- 当A链上的NFT远离A的时候，nftTransfer模块在A链上托管现有的NFT资产，并在接收链B上mintNFT。
- 当B链上的NFT靠近A的时候，nftTransfer模块会在B链上销毁本地凭证，并在接收链A上释放NFT资产托管。
- Acknowledgement 数据用于处理失败，例如无效的目标帐户。 返回失败确认比中止交易更可取，因为它更容易使发送链能够根据失败的性质采取适当的行动。

`createOutgoingPacket `必须由模块中的交易处理程序调用，该处理程序执行适当的签名检查，特定于主机状态机上的帐户所有者。

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
            // 托管NFT（如果余额不足，则假设失败）
            nft.lock(sender, escrowAccount, class, id, uri)
        } else {
            // 接收者靠近源链 则burnNFT 
            nft.BurnNFT(sender, class, id)
        }
        NonFungibleTokenPacketData data = NonFungibleTokenPacketData{class, id, uri, awayFromOrigin, sender, receiver}
        handler.sendPacket(Packet{sequence, port, relayChain, destChain})
    }
```

当路由模块收到发往该模块的数据包时，路由模块会调用`onRecvPacket`。

```go
function onRecvPacket(packet: Packet) {
    NonFungibleTokenPacketData data = packet.data
    // 构建默认的成功确认
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

当路由模块发送的数据包被确认时，onAcknowledgePacket 被路由模块调用。

```go
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // 如果转账失败，退还nft
  if (!ack.success)
    refundTokens(packet)
}
```

refundTokens 由 onAcknowledgePacket 调用，以将托管NFT退还给原始发送者。

```go
function refundTokens(packet: Packet) {
  NonFungibleTokenPacketData data = packet.data
  if awayFromSource {
    // 将托管的NFT再发回给发送者
    nft.unlock(escrowAccount, data.receiver, class,id)
  } else {
    // 如果是靠近源链，则mintNFT
    nft.MintNft(class, id, nftName, uri, nftData, sender)
  }
}
```

## 跨链单跳举例

### A链发送NFT到B链

>  从A链发送NFT到B链，中继链指定为空，A链只需要将NFT托管，B链生成一个对应的NFT



A链需要将NFT托管，然后构造一个跨链数据包发送到B链

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

B链会`mint`一个新的`NFT`，**newClass = prefix + "/" + sourceChain + "/" + class = tibc/nft/A/hellokitty**


### B链将NFT退回到A

> 将nft从B链退回到A链，B链burnNFT ,A 链释放托管的NFT



B链需要将要退回的NFT进行burn操作，然后构造一个跨链数据包

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

A链会取消HelloKitty的托管。



## 跨链两跳举例

### A链通过B链将NFT发送到C

> 首先A托管此class的NFT，然后再B链上mint一个nft，nft的`class`为`tibc/nft/A/hellokitty`，接着B会托管此NFT并生成一个新的packet，目标链指向C，C会再mint对应的nft。

A链需要将NFT托管，然后构造一个跨链数据包发送到B链

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

B链会mint一个新的NFT，`newClass` = prefix + "/" + sourceChain + "/" + class = `tibc/nft/A/hellokitty`,接着B链会托管刚生成的新的NFT，然后重新构造新的数据包：

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

C链会mint一个新的NFT，如果已经有prefix的话，则newClass = packet.class + packet.sourceChain = tibc/nft/A/B/hellokitty



### C链通过B链将NFT退回到A链

> C上的nft再退回到A链的过程：C会判断自己是不是远离源，如果不是，则burnNFT并构建一个packet，class为tibc/nft/A/B/hellokitty（A-C的完整路径），接着到了B之后会释放掉B托管的NFT，并burnNFT(class=tibc/nft/A /hellokitty),最后会再构造一个packet，让A链去释放最初托管的NFT，从而达到从C-A的退回目的。

C链判断是否远离源链，然后burn NFT，之后构造packet发到B链


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

B链接受到C链发的包后，会根据包去掉packet.descChain，生成一个newClass,并到自己链上查询，如果有的话，直接就取消托管，并且burn掉。之后会构造新的包发给A。


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

A接收到包后会判断是否远离源链，并去掉前缀，查看自己链上是否存在此NFT，并释放掉。

## 向后兼容

不适用。

## 向前兼容

## 示例实现

不久到来。

## 其他实现

不久到来。

## 历史

## 版权

本文中的所有内容均在 Apache 2.0 下获得许可。



