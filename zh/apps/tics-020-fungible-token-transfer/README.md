| tics | title                                      | stage  | category                        | kind       | requires  | required-by     | author                               | created  | modified  |
| ---------------- | ---------------------------------------------- | ---------- | ----------------------------------- | -------------- | ------------- | --------------------- | ---------------------------------------- | ------------ | ------------- |
| 18               | Fungible Token Transfer | draft  | TIBC/TAO  | interface  | 23            |  | | 2021-07-26   | 2021-07-26   |

## 概要

本标准文档规定了数据包数据结构、状态机处理逻辑和编码细节，用于在不同链上的两个模块之间通过 TIBC 端口传输可替代 token。


### 动机

通过 TIBC 协议连接的一组链的用户可能希望利用在另一个链上的一个链上发行的资产，也许是为了利用诸如交换或隐私保护之类的附加功能，同时保留发行链上原始资产的可替代性。本应用层标准描述了一种在与 TIBC 相连的链之间传输可替代 token 的协议，该协议保持了资产的可替代性，保留了资产所有权，限制了拜占庭式错误的影响，并且不需要额外的许可。

### 定义

TIBC 处理程序接口和 TIBC 路由模块接口分别在 [TICS 04](../../core/tics-004-port-and-packet-semantics) 和 [TICS 26](../../core/tics-026-packet-routing) 中定义。

### 所需属性 

- 保留可替代性（双向peg）。
- 保持总供应量（单个源链和模块上的是恒定的或通胀的）。
- 容错：防止由于链 B 的拜占庭行为导致源自链 A 的 token 的拜占庭式膨胀（尽管任何将 token 发送到链 B 的用户都可能面临风险）。

### 技术指标

#### 数据结构

只需要一种数据包数据类型：FungibleTokenPacketData，它指定 denom ， amount ，发送帐户和接收帐户。

```typescript
interface FungibleTokenPacketData {
  denomination: string
  amount: uint256
  sender: string
  receiver: string
}
```

当使用 `TICS 20` 协议跨链发送 token 时， token 开始累积其已传输过的记录。 该信息被编码到 `denomination` 字段中。

`denomination` 字段以 `{prefix}/{sourceChain}/{denom}` 的形式实现，其中 `prefix = "tibc/ft/"` 。 当字段中没有 `prefix` 和 `sourceChain` 时，发送链为FT的源链； 如果该字段包含一个 `prefix` 和一个 `sourceChain` ，则FT从 `sourceChain` 转移。 如在 `denomination` 是 `tibc/ft/A/B/denomination` 的情况下，如果 FT 现在存在于链 `C` 上，则 FT 从链 `A` 转移到链 `B` ，然后转移到链 `B` 链 `C` 。 如果链  `C` 希望将FT返回到它的原始链，它必须先将 FT 返回到链 `B` 以将其更新为 `tibc/ft/A` ，然后再通过链 `B` 回到链 `A`。

当从 FT 的源链传输到目标链或从目标链返回到源链时，指定 `awayFromOrigin` 。在跳中构造的数据包的 `awayFromOrigin` 字段是接收到的数据包中的 `awayFromOrigin` 字段数据。

确认数据类型描述传输是成功还是失败，以及失败的原因（如果有）。

```typescript
type FungibleTokenPacketAcknowledgement = FungibleTokenPacketSuccess | FungibleTokenPacketError;

interface FungibleTokenPacketSuccess {
  // This is binary 0x01 base64 encoded
  success: "AQ=="
}

interface FungibleTokenPacketError {
  error: string
}

```

### 子协议

此处描述的子协议应在“FT传输桥”模块中实现，该模块可访问 FT 模块和 Packet 模块。

#### 端口设置

必须在创建模块时（可能在初始化区块链本身时）调用 `setup` 函数一次以绑定到适当的端口并创建托管地址（由模块拥有）。

```typescript
function setup() {
  capability = routingModule.bindPort("ftTransfer", ModuleCallbacks{
    onRecvPacket,
    onAcknowledgePacket,
    onCleanPacket
  })
}

```

一旦调用了 setup 函数，跨链数据可以通过 TIBC 路由模块在不同链上的 ftTransfer 模块实例之间传输。

#### 路由模块回调

在链 A 和链 B 之间：

- 当链 `A` 上的 `FT` 远离链 `A` 时，`ftTransfer` 模块在链 `A` 上托管现有的 `FT` 资产，并在目标链链 `B` 上 `mintFT`。
- 当链 `B` 上的 `FT` 靠近链 `A` 的时候，`ftTransfer` 模块会在链 `B` 上销毁本地`FT` ，并在源链链 `A` 上释放 `FT` 资产托管。
- `Acknowledgement` 数据用于处理失败，例如无效的 `denomination` 或无效的目标帐户。 返回失败确认比中止交易更可取，因为它更容易使发送链能够根据失败的性质采取适当的行动。

`createOutgoingPacket` 必须由模块中的交易处理程序调用，该模块执行适当的签名检查，主要针对主机状态机上的帐户所有者。

```typescript

function createOutgoingPacket(
    denomination: string,
    amount: uint256,
    sender: string,
    receiver: string,
    awayFromOrigin: boolean,
    destChain:string,  
    relayChain:string) {

  if awayFromOrigin {
    ft.lock(sender, denomination, amount)
  } else {
    // receiver is source chain, burn FT
    ft.burn(sender, denomination, amount)
  }
  FungibleTokenPacketData data = FungibleTokenPacketData{denomination, amount, sender, receiver}
  handler.sendPacket(Packet{sequence, port, relayChain, destChain, data})
}
```

当收到发往该模块的数据包时，路由模块调用 `onRecvPacket` 。

```typescript
function onRecvPacket(packet: Packet) {
  FungibleTokenPacketData data = packet.data
  // construct default acknowledgement of success
  FungibleTokenPacketAcknowledgement ack = FungibleTokenPacketAcknowledgement{true, null}
  if awayFromOrigin {
 
  } else {

  }
  return ack
}
```

