| tics | title                   | stage | category | kind          | requires | author                           | created    | modified   |
| ---- | ----------------------- | ----- | -------- | ------------- | -------- | -------------------------------- | ---------- | ---------- |
| 4    | Port & Packet Semantics | draft | TIBC/TAO | instantiation | 2, 3, 26 | Wenxi Cheng <vincent@bianjie.ai> | 2021-07-23 | 2021-07-26 |

## 大纲

`Port`规定了端口分配系统，通过该系统，模块可以绑定到 TIBC 处理程序分配的唯一命名的端口。然后可以在端口间传递`Packet`，并且可以通过最初绑定到它们的模块进行传输或之后释放。

`Packet`定义了链间数据包标准。发送和接收 TIBC 数据包的模块决定如何构造数据包数据以及如何处理传入的数据包数据，并且必须使用自己的应用程序逻辑根据数据包包含的数据来决定应用哪种状态交易。

### 动机

区块链间通信协议使用跨链消息传递模型。IBC 数据包通过外部relayer进程从一个区块链中继到另一个区块链。链 `A` 和链 `B` 独立地确认新的块，从一个链到另一个链的数据包可能被延迟、审查或任意重新排序。数据包对于relayer是可见的，任何relayer进程可以从一个区块链读取和提交到任何其他区块链。

TIBC 协议必须提供准确一次的传递保证，以允许应用程序推断两个链上连接模块的组合状态。例如，一个应用程序可能希望允许一个标记化的资产之间转移，并在多个区块链上持有，同时保持可替换性和供应保护。当一个特定的 IBC 包被提交给链 `B` 时，应用程序可以在链 `B` 上铸造资产凭证，并要求发出请求的链 `A` 上的托管相等数量的资产，直到这些凭证随后被返回到链 `A`，并且在相反的方向上有一个 IBC 数据包。这种顺序保证以及正确的应用逻辑可以确保两条供应链的总供应量都得到保留，而且在 b 链上铸造的任何凭证日后都可以兑换回 `A` 链。

### 定义

`ConsensusState `符合 [TICS 2 ](../yics-002-client-semantics)中的定义。

`Port` 是一种特殊的标识符，用于对模块进行权限通道打开和使用。

```go
enum Port {
  FT,
  NFT,
  SERVICE,
  CONTRACT,
}
```

`module` 是独立于 TIBC 处理程序的主机状态机的子组件。例子包括 Ethereum 智能合同和 Cosmos SDK & 基本模块。TIBC 规范除了主机状态机使用对象能力或源身份验证对模块的权限端口的能力之外，没有对模块功能进行任何设想。

`hash` 是一种通用的抗冲突散列函数，其细节必须由使用该通道的模块商定。`hash`可以由不同的链来定义。

 `Packet` 是一个特定的接口，定义了跨链数据包规范：

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

- `sequence` 编号对应于发送的顺序
- `port` 标识对应的数据包发送和接收的端口。
- `sourceChain` 标识数据包的发送链。
- `destChain`标识数据包的接受链。
- `relaychain`标识数据包途经的中继链，为空表示不经由中继链。
- `data` 是一个不透明的值，可以由相关模块的应用程序逻辑定义。

> 请注意，`Packet` 永远不会直接序列化。 相反，它是在某些函数调用中使用的中间结构，可能需要由调用 TIBC 处理程序的模块创建或处理。

> `Openepacket` 是一个数据包，但由主机状态机隐藏在模糊数据类型中，因此模块只能将其传递给 TIBC 处理程序，而不能对其进行操作。TIBC 处理程序可以将一个 Packet 强制转换为一个 OpaquePacket，反之亦然。

```go
type OpaquePacket = object
```

`CleanPacket`定义了对状态清理的跨链数据包：

```go
interface CleanPacket {
  sequence: uint64
  sourceChain: Identifier
  destChain: Identifier
  relayChain: Identifier
}
```

- `sequence` 标识待清除数据的最大编号
- `sourceChain` 标识数据包的发送链。
- `destChain`标识数据包的终止链。
- `relaychain`标识数据包途经的中继链，可为空。

> 请注意，`CleanPacket`执行操作是幂等的，因此自身不包括sequence属性用于避免重复执行的操作。

### 所需属性

#### 效率

- 数据包传输和确认的速度应该仅仅受到底层链路的速度的限制。在可能的情况下，证明应该是成批的。

#### 一次传输

- 在通道一端发送的 TIBC 数据包应该准确地传送到另一端一次
- 如果链路中的一个或两个中断，数据包可能不超过一次传递，一旦链路恢复数据包应该能够再次流动

#### 授权

## 技术规格

### 数据流可视化

### 初步报告

#### 存储路径

`nextSequenceSend`无符号整数计数器存储，标识下一个跨链包的序号：

```go
function nextSequenceSendPath(sourceChainIdentifier: Identifier, destChainIdentifier: Identifier): Path {
  return "seqSends/{sourceChainIdentifier}/{destChainIdentifier}/nextSequenceSend"
}
```

对分组数据字段的固定大小的承诺存储在`packetCommitmentPath`下：

```go
function packetCommitmentPath(sourceChainIdentifier: Identifier, destChainIdentifier Identifier, sequence: uint64): Path {
    return "commitments/{sourceChainIdentifier}/{destChainIdentifier}/packets/" + sequence
}
```

数据包收据数据存储在 `packetReceiptPath` 下：

```go
function packetReceiptPath(sourceChainIdentifier: Identifier, destChainIdentifier Identifier, sequence: uint64): Path {
    return "receipts/{sourceChainIdentifier}/{destChainIdentifier}/receipts/" + sequence
}
```

数据包确认数据存储在 `packetAcknowledgementPath` 下:

```go
function packetAcknowledgementPath(sourceChainIdentifier: Identifier, destChainIdentifier Identifier, sequence: uint64): Path {
    return "acks/{sourceChainIdentifier}/{destChainIdentifier}/acknowledgements/" + sequence
}
```

对历史数据清理的的存储在`packetCleanPath`下：

```go
function packetCleanPath(chainIdentifier: Identifier): Path {
    return "cleans/{chainIdentifier}/clean"
}
```

### 子协议

#### 标识符验证

#### 数据包的流动和处理

##### 包生命周期

要将数据包从机器 `A` 的模块1发送到机器 `B` 的模块 *2* ，必须从头开始执行以下步骤。

该模块可以通过 [TICS 26 ](../tics-026-routing-module) 与 TIBC 处理程序连接。

##### 发送数据包

模块调用 sendPacket 函数，以便将调用模块所拥有的通道端上的 TIBC 数据包发送到对应方链上的相应模块。

调用模块必须与调用 `sendPacket` 一起自动地执行应用程序逻辑。

TIBC 处理程序按顺序执行以下步骤:

- 检查目标链client是否可用
- 检查调用模块是否拥有发送端口
- 增加与通道关联的发送序列计数器
- 存储对数据包数据和数据包超时的固定大小承诺

请注意，完整的数据包并不存储在链的状态中——仅仅是对 data & timeout 值的一个简短的哈希提交。分组数据可以从交易执行中计算出来，并可能作为relayer可以索引的日志输出返回。

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

#### 接收数据包

一个模块调用 `recvPacket` 函数，以接收在对应方链的相应信道端发送的 TIBC 包。

与调用 `recvPacket` 相结合，调用模块必须执行应用程序逻辑或将数据包排队以便将来执行。

TIBC 处理程序按顺序执行以下步骤:

- 检查来源链client是否可用
- 检查调用模块是否拥有接收端口
- 检查输出链状态中数据包数据承诺的包含证明
- 设置存储路径以指示已收到数据包
- 增加与通道结束关联的数据包接收序列

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

#### 写确认

模块调用 `writeacknowledging` 函数，以便写入处理 TIBC 数据包所产生的数据，然后发送链可以验证这些数据，这是一种“执行收据”或“ RPC 调用响应”。

调用模块必须与调用 `writeacknowledging` 一起自动地执行应用程序逻辑。

这是一个异步确认，只有在处理完成时，才需要在接收到数据包时确定其内容。在同步情况下，可以在与 `recvPacket` 相同的事务中(原子地)调用 `writeAcknowledgement`。

不需要确认数据包; 但是，如果已排序的通道使用确认，则必须确认所有数据包或不确认数据包(因为确认是按顺序处理的)。请注意，如果数据包没有得到确认，则无法在源链上删除数据包提交。TIBC 的未来版本可能包括一些方法，让模块指定它们是否会确认数据包，以便进行清理。

`Writetacknowledging` 不检查正在被确认的数据包是否真正被接收，因为这将导致对确认的数据包进行两次验证。这方面的正确性是调用模块的责任。调用模块必须只使用以前从 `recvPacket` 接收到的数据包调用 `writebackiding`。

TIBC 处理程序按顺序执行以下步骤:

- 检查此数据包的确认是否尚未写入
- 在数据包唯一的存储路径上设置不透明确认值

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

#### 处理确认

`acknowledgePacket ` 函数由一个模块调用，以处理由调用模块先前发送到对应方链上的对应方模块的数据包的确认。`acknowledgePacket `还清理数据包承诺，由于已经收到数据包并对其进行操作，因此不再需要数据包承诺。

调用模块可以结合调用 `acknowledgePacket ` 自动执行适当的应用程序确认处理逻辑。

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

##### 确认规范

在 TIBC 协议中，从远程链返回的确认被定义为任意字节。此数据可能编码成功执行或失败(除了超时之外的任何事情)。没有通用的方法来区分这两种情况，这要求任何客户端数据包可视化程序理解每个应用程序特定的协议，以区分成功或失败的中继情况。为了减少这个问题，我们提供了一个附加的确认格式规范，应该由应用程序特定的协议使用。

```
message Acknowledgement {
  oneof response {
    bytes result = 21;
    string error = 22;
  }
}
```

#### 清理状态

必须确认数据包才能进行清理。

定义一个新的状态清理packet用于清理跨链数据包生命周期中产生的数据存储。该packet可以清理自身的存储。

##### 发送清理 Packet

模块调用`sendCleanPacket`函数，以便将调用模块所拥有的通道端上的 TIBC 数据包发送到对应方链上。

```go
function sendCleanPacket(packet: CleanPacket) Packet{
    provableStore.set(packetCleanPath(packet.sourceChain), packet.sequence)

    // log that a packet has been sent
    emitLogEntry("CleanPacket", {sourceChain: packet.sourceChain, destChain: packet.destChain, relayChain: packet.relayChain, sequence: packet.sequence)
    return packet
}
```

##### 接收清理数据包

一个模块调用 `recvCleanPacket` 函数，以接收在对应方链的相应信道端发送的 TIBC 包。

与调用 `recvCleanPacket` 相结合，调用模块必须执行应用程序逻辑或将数据包排队以便将来执行。

TIBC 处理程序按顺序执行以下步骤:

- 检查来源链client是否可用
- 设置存储路径以指示已收到清理数据包
- 清理sequence小于指定值的`Receipt`和`Acknowledgement`

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
