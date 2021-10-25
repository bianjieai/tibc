| ics | title                                      | stage  | category                        | kind       | requires  | required-by     | author                               | created  | modified  |
| ---------------- | ---------------------------------------------- | ---------- | ----------------------------------- | -------------- | ------------- | --------------------- | ---------------------------------------- | ------------ | ------------- |
| 24               | Host Environments | draft  | IBC/TAO  | interface  | 23            |  | | 2021-07-21   | 2021-07-21   |

## Synopsis 大纲

本规范定义了必须提供的接口的最小集合，以及承载 链间通信协议 实现的状态机必须满足的属性。

### Motivation 动机

tIBC 被设计成一个通用标准，将由各种区块链和状态机托管，并且必须明确定义主机的要求。

### Definitions 定义

### Desired Properties 期望的特性

tIBC 应该要求底层状态机提供尽可能简单的接口，以最大限度地提高正确实现的易用性。

## Technical Specification 技术规格

### Module system 模块系统


主机状态机必须支持模块系统，通过该系统，独立的、可能相互不信任的代码包可以安全地在同一个账本上执行，控制它们如何以及何时允许其他模块与它们通信，并由“ master 模块”或执行环境识别和操作。

tIBC/TAO规范定义了两个模块的实现：核心的“tIBC handler”模块和“tIBC relayer”模块。tIBC/APP 规范进一步定义了用于特定分组处理应用程序逻辑的其他模块。tIBC 要求可以使用“ master 模块”或执行环境来授予主机状态机上的其他模块对 tIBC 处理程序模块 和/或 tIBC routing 模块的访问权限，但在其他方面，不会对可能位于状态机上的任何其他模块的功能或通信能力提出要求。

### Paths, identifiers, separators 路径，标识符，分隔符

`Identifier ` 是一个 bytestring，用作存储在状态中的对象（如连接、通道或轻客户端）的键。

标识符必须为非空（长度为正整数）。

标识符只能由以下类别之一的字符组成：

- 字母和数字数字
- `. `, `_ `, `+ `, `- `, `#`
- `[ `, `] `, `< `, `>`

`Path` 是一个bytestring，用作状态中存储的对象的键。path 只能包含标识符、常量字符串和分隔符`/`。

标识符并不打算成为有价值的资源 -- 为了防止名称占用，可以实现最小长度要求或伪随机生成，但本规范没有施加特定的限制。

分隔符 `/` 用于分隔和连接两个标识符或一个标识符和一个常量bytestring。标识符不能包含 `/` 字符，这样可以防止歧义。

变量插值，用大括号表示，在本规范中用作定义路径格式的缩写，例如  `client/{clientIdentifier}/consensusState `

除非另有规定，否则本规范中列出的所有标识符和所有字符串必须编码为ASCII。

默认情况下，标识符的最小和最大字符长度如下:

| Port identifier | Client identifier 客户标识符 |
| -------------------------- | ---------------------------- |
| 2 - 64             | 9 - 64 9-64                  |

### Key/value Store 键/值存储区

主机状态机必须提供一个 key/value 存储接口，该接口具有三个以标准方式运行的功能：

```
type get = (path: Path) => Value | void
type set = (path: Path, value: Value) => void
type delete = (path: Path) => void
```

`Path ` 如上所述。 `Value ` 是特定数据结构的任意bytestring编码。编码细节留给单独的ics处理。

这些函数必须只允许IBC处理程序模块（其实现在单独的标准中描述）使用，因此只有IBC处理程序模块才能`set`  或 `delete ` 可由 `get` 读取的路径。这可以实现为整个状态机使用的较大 key/value 存储的子存储（前缀键空间）。

主机状态机必须提供此接口的两个实例—一个 `provableStore` 用于其他链读取的存储（即证明给其他链读取的存储），另一个 `privateStore` 用于主机本地存储，可以在其上调用 `get` 、 `set` 和 `delete` ，例如 `provableStore.set('some/path', 'value') ` 。

`provableStore`:

- 必须写入键/值存储，其数据可通过 [ICS 23] 中定义的向量承诺进行外部证明
- 必须使用这些规范中提供的规范数据结构编码作为proto3文件

`privateStore`:

- 可能支持外部证明，但不是必需的 -- tIBC 处理程序永远不会向其写入需要证明的数据。
- 可以使用规范的 proto3 数据结构，但不是必需的 -- 它可以使用应用程序环境首选的任何格式。

> 注意：任何提供这些方法和属性的键/值存储接口对于 tIBC 来说都是足够的。主机状态机可以使用 path 和 value 映射来实现 “代理存储” ，path 和 value 映射与通过存储接口设置和检索的 path 和 value 对不直接匹配- path 可以分组到存储在页面中的 bucket 和 value 中，这些 bucket 和 value 可以在单个承诺中得到证明，path-spaces 可以以某种双射方式等非连续地重新映射，只要 `get`, `set`, 和 `delete` 的行为符合预期，并且其他机器可以在可证明存储中验证 path 和 value 对（或其不存在）的承诺证明。如果适用， store 必须在外部公开这个映射，以便客户（包括 relayer ）可以确定 store 的设计 以及如何构造证明。使用这种代理存储的机器的客户端也必须理解映射，因此它将需要新的客户端类型或参数化的客户端。

> 注意：此接口不需要任何特定的存储后端或后端数据设计。状态机可以选择使用根据其需要配置的存储后端，只要上面的存储满足指定的接口并提供承诺证明。

### Path-space 路径空间

目前， tIBC/TAO 为 `provoblestore` 和 `privateStore`  推荐以下路径前缀。

未来的 path 可能会在协议的未来版本中使用，因此可证明存储中的整个 key-space 必须为 tIBC 处理程序保留。

只要本文定义的 key 格式与机器实现中实际使用的 key 格式之间存在 二分映射（二分图），可证明存储中使用的 key 就可以在每个客户端类型的基础上安全地变化。

只要 tIBC handler 对所需的特定 key 具有独占访问权，私有存储的部分内容就可以安全地用于其他目的。只要在本文定义的 key 格式和在私有存储实现中实际使用的密钥格式之间存在 二分映射，在私有存储中使用的 key 就可以安全地变化。

请注意，下面列出的与客户机相关的路径反映了[ICS 7] 中定义的 Tendermint 客户端，对于其他客户端类型可能有所不同。

| Store          | Path format                                                                    | Value type        | Defined in |
| -------------- | ------------------------------------------------------------------------------ | ----------------- | ---------------------- |
| provableStore  | "clients/{identifier}/clientType"                                              | ClientType        | ICS 2 |
| privateStore   | "clients/{identifier}/clientState"                                             | ClientState       | ICS 2 |
| provableStore  | "clients/{identifier}/consensusStates/{height}"                                | ConsensusState    | ICS 7 |
| privateStore   | "ports/{identifier}"                                                           | CapabilityKey     | ICS 5 |



### Module layout 模块布局

在空间上表示，模块的布局及其在主机状态机上包含的规范如下所示（Aardvark、Betazoid和Cephalopod是任意模块）：

```
+----------------------------------------------------------------------------------+
|                                                                                  |
| 主机状态机                                                              |
|                                                                                  |
| +-------------------+       +--------------------+      +----------------------+ |
| | Module Aardvark   | <-->  | tIBC Routing Module|      | IBC Handler Module   | |
| +-------------------+       |                    |      |                      | |
|                             | Implements ICS 26. |      | Implements ICS 2, 5  | |
|                             |                    |      | internally.          | |
| +-------------------+       |                    |      |                      | |
| | Module Betazoid   | <-->  |                    | -->  | Exposes interface    | |
| +-------------------+       |                    |      | defined in ICS 25.   | |
|                             |                    |      |                      | |
| +-------------------+       |                    |      |                      | |
| | Module Cephalopod | <-->  |                    |      |                      | |
| +-------------------+       +--------------------+      +----------------------+ |
|                                                                                  |
+----------------------------------------------------------------------------------+
```

### Consensus state introspection 共识状态的反省


主机状态机必须提供使用 `getCurrentHeight` 内省其当前高度的能力：

```
type getCurrentHeight = () => Height
```

主机状态机必须定义一个满足 ICS 2 要求的唯一 `consenssusstate` 类型，并使用规范的二进制序列化。

主机状态机必须提供用 `getConsensusState ` 内省自己一致性状态的能力：

```
type getConsensusState = (height: Height) => ConsensusState
```

`getConsensusState` 必须返回至少一些连续最近高度的 `n` 的 共识状态，其中 `n` 对于主机状态机是常量。早于 `n` 的高度可能会被安全地修剪（会导致未来对这些高度的调用失败）。

主机状态机必须提供使用 `getStoredRecentConsensusStateCount` 内省此存储的最近一致性状态计数 `n` 的功能：

```
type getStoredRecentConsensusStateCount = () => Height
```

### Commitment path introspection 承诺路径内省

主机链必须提供使用 `getCommitmentPrefix` 检查其提交 path 的能力：

```
type getCommitmentPrefix = () => CommitmentPrefix
```

结果 `CommitmentPrefix` 是主机状态机的 key/value 存储使用的前缀。对于主机状态机的 `CommitmentRoot` 根 和 `CommitmentState` 状态 ，必须保留以下属性：

```
if provableStore.get(path) === value {
  prefixedPath = applyPrefix(getCommitmentPrefix(), path)
  if value !== nil {
    proof = createMembershipProof(state, prefixedPath, value)
    assert(verifyMembership(root, proof, prefixedPath, value))
  } else {
    proof = createNonMembershipProof(state, prefixedPath)
    assert(verifyNonMembership(root, proof, prefixedPath))
  }
}
```

对于主机状态机，`getCommitmentPrefix` 的返回值必须是常量。

### Timestamp access 时间戳访问

主机链必须提供一个当前 Unix 时间戳，可通过 `currentTimestamp` 访问：

```
type currentTimestamp = () => uint64
```

为了在超时中安全地使用时间戳，后续报头中的时间戳必须是非递减的。

### Port system  端口系统

主机状态机必须实现 端口系统，其中 tIBC 处理程序可以允许主机状态机中的不同模块绑定到唯一命名的端口。端口由 `Identifier` 标识。

主机状态机必须实现与 tIBC 处理程序的权限交互，以便：

- 一旦某个模块绑定到某个端口，在该模块释放该端口之前，其他模块都不能使用该端口
- 一个模块可以绑定到多个端口
- 端口分配为先到先服务，当状态机首次启动时，可以绑定已知模块的“保留”端口

这种授权可以通过每个端口的唯一引用（对象功能）（Cosmos SDK的la）、源身份验证（Ethereum）或其他访问控制方法来实现，在任何情况下都是由主机状态机强制实现的。详见 ICS 5 。

### Datagram submission 数据报提交

实现路由模块的主机状态机可以定义一个 `submitDatagram` 函数，将交易中包含的 数据报 直接提交给路由模块（在 ICS 26 中定义）：

```
type submitDatagram = (datagram: Datagram) => void
```
`submitDatagram` 允许 relayer 将 tIBC 数据报直接提交到主机状态机上的路由模块。主机状态机可能要求提交数据报的 relayer 有一个支付交易费用的帐户，在更大的交易结构中对数据报进行签名，等等。 `submitDatagram` 必须定义和构造所需的任何此类打包。

### Exception system 异常系统

主机状态机必须支持异常系统，通过该系统，交易可以中止执行并还原以前所做的任何状态更改（包括同一交易中发生的其他模块中的状态更改），不包括消耗的 gas 和适当的费用支付，并且系统不变违规 可以停止状态机。

此异常系统必须通过两个函数公开： `aborttransactionexcelless` 和 `abortSystemUnless` ，前者还原交易，后者停止状态机。

```
type abortTransactionUnless = (bool) => void
```

如果传递给 `aborttransactionuncell` 的布尔值为 `true` ，则主机状态机无需执行任何操作。如果传递给 `aborttransactionuncell` 的布尔值为 `true` ，则主机状态机必须中止交易并还原以前所做的任何状态更改，不包括消耗的 gas 和适当的费用支付。

```
type abortSystemUnless = (bool) => void
```
如果传递给 `abotsystemunless` 的布尔值为 `true` ，则主机状态机无需执行任何操作。如果传递给 `abortSystemUnless` 的布尔值为 `true` ，则主机状态机必须停止。

### Data availability 数据可用性

为了交付或超时安全，主机状态机必须具有最终的数据可用性，这样状态中的任何 key/value对 最终都可以由 relayer 检索。为了安全起见，不需要数据可用性。

对于数据包中继的活跃度，主机状态机必须具有有限的交易活跃度（因此必须具有共识活跃度），以便传入交易在块高度范围内得到确认（特别是，小于分配给数据包的超时）。

IBC 数据包，以及其他不直接存储在状态向量中但 relay 依赖的数据，必须可供 relayer 有效地计算。

特定共识算法的轻客户端可能有不同 和/或 更严格的数据可用性要求。

### Event logging system 事件记录系统

主机状态机必须提供一个事件记录系统，在交易执行过程中可以记录任意数据，这些数据可以存储、索引，然后由执行状态机的进程查询。relayer 利用这些事件日志来读取 tIBC 数据包数据和超时，这些数据包数据和超时不是直接以链状态存储的（因为这种存储被认为是昂贵的），而是通过简洁的加密承诺（只存储承诺）。 

该系统至少应具有一个用于发送日志条目的函数和一个用于查询过去日志的函数，大致如下所示。

在交易执行期间，状态机可以调用函数 `emitLogEntry` 来写入日志项：

```
type emitLogEntry = (topic: string, data: []byte) => void
```
`queryByTopic` 函数可由外部进程（如 relayer ）调用，以检索在给定高度执行的 tx 所编写的与给定主题相关联的所有日志条目。

```
type queryByTopic = (height: Height, topic: string) => []byte[]
```

还可以支持更复杂的查询功能，并且可以允许更高效的 relayer 查询，但这不是必需的。

### Handling upgrades 处理升级

主机可以安全地升级其状态机的某些部分，而不会中断 tIBC 功能。为了安全地做到这一点， tIBC 处理程序逻辑必须与规范保持一致，并且所有 tIBC 相关的状态（在 provable 和 private 存储中）必须在整个升级过程中保持不变。如果在其他链上存在用于升级链的客户端，并且升级将更改轻客户端验证算法，则必须在升级之前通知这些客户端，以便它们可以安全地原子切换并保持连接和通道的连续性。

## Backwards Compatibility 向后兼容性

Not applicable.

不适用。

## Forwards Compatibility 向前兼容性


## Example Implementation 实施范例


## Other Implementations 其他实现


## History 历史



## Copyright 版权所有
