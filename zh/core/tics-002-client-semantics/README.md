
| tics | title            | stage | category | kind      | requires | author              | created    | modified   |
| ---- | ---------------- | ----- | -------- | --------- | -------- | ------------------- | ---------- | ---------- |
| 2    | Client Semantics | draft | TIBC/TAO | interface | 23,24    | zhiqiang@bianjie.ai | 2021-07-23 | 2021-07-26 |

## 大纲

本文描述了轻客户端的整个生命周期，主要包括：创建、更新、升级以及利用轻客户端状态验证来源链数据的整个流程的定义，每种轻客户端的详细实现不在本文档的范围之内。

### 动机

在TIBC协议中，参与者（可能是直接用户，链下进程或一台机器）需要能够验证另一台机器的共识算法已同意的状态更新，并拒绝另一台机器的共识算法尚未达成共识的任何可能的更新。轻客户端是机器可以执行的算法。该标准规范了轻客户端模型和需求，因此只要提供满足列出要求的相关轻客户端算法，TIBC协议就可以轻松地与运行新的共识算法的新机器集成

### 定义

- `get`、`set`、`Path` 和`Identifier` 在 TICS 24 中定义。
- `CommitmentRoot` 在 TICS 23 中定义。 它必须为下游逻辑提供一种廉价的方式来验证键/值对是否处于特定高度的状态。
- `ConsensusState` 是表示有效性断言状态的接口类型。`ConsensusState`必须能够验证相关共识算法同意的状态更新。它也必须以规范的方式可序列化，以便第三方（例如对方机器）可以检查特定机器是否已存储特定机器的共识状态。它最终必须由它所针对的状态机进行自省，以便状态机可以在过去的高度查找自己的共识状态。
- `ClientState`是代表客户端状态的接口类型。`ClientState`必须公开查询函数以验证特定高度状态中键/值对的成员资格或非成员资格，并检索当前的ConsensusState。

### 所需属性

轻客户端必须提供一种安全算法来验证其他链的规范`header`，使用现有的`ConsensusState`。 然后，更高级别的抽象将能够使用存储在`ConsensusState`中的 `CommitmentRoot`来验证状态的子组件，这些子组件保证已由其他链的共识算法提交。

有效性断言应反映运行相应共识算法的全节点的行为。 给定一个`ConsensusState`和一个消息列表，如果一个完整节点接受由`Commit`生成的新的`Header`，那么轻客户端也必须接受它，如果一个完整节点拒绝它，那么轻客户端也必须拒绝它。

轻客户端不会重放整个消息副本，因此在共识不当行为的情况下，轻客户端的行为可能与完整节点的行为不同。 在这种情况下，可以生成一个不当行为证明，证明有效性断言和完整节点之间的差异，并提交给链，以便链可以安全地停用轻客户端，使过去的状态根无效，并等待更高级别的干预。

## 技术规格

### 数据存储

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

`ChainName` 指定了链的名称，必须在genesis中指定，链初始化(`InitGenesis`)时写入区块链， 并且它必须是唯一的，用于在其他链上创建自己的轻客户端。

```go
func SetChainName(chainName string) {
    store := ctx.KVStore(k.storeKey)
    store.Set("client/chainName", chainName)
}
```

### 数据结构

#### 客户端类型

客户端类型指明了创建的轻客户端所使用的的共识算法类型，为了方便以后扩展，定义为常量形式，定义如下：

```golang
type ClientType = string
```

#### 区块头

`区块头`是由客户端类型定义的接口数据结构，它提供更新`ConsensusState`的信息。 可以将`区块头`提交给关联的客户端以更新存储的`ConsensusState`。 它们可能包含一个高度、一个证明、一个承诺根，还可能包含对有效性断言的更新。不同的区块链实现方式各有不同，所以区块头被定义为接口形式：

```golang
type Header interface {
    ClientType() string
    GetHeight() Height
    ValidateBasic() error
}
```

#### 高度

`高度`定义了目标链当前的区块高度信息，考虑到分叉的可能性还应该包含一些其他信息，比如`版本信息`等，定义如下：

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

#### 状态

`状态`定义了当前轻客户端的所处的状态，例如`活跃`、`过期`、`未知`，定义如下：

```go
    type Active  = "Active"
    type Expired = "Expired"
    type Unknown = "Unknown"
```

具体的状态判断逻辑由具体轻客户端实现，当轻客户端过期后，无法直接更新，只能采用提议升级的方式更新轻客户端状态。

#### 客户端状态

`客户端状态`是由客户端类型定义的接口类型的数据结构。它可以保持任意内部状态，以跟踪经过验证的根和过去的不良行为，客户端状态由一系列接口组成，共同完成跨链数据包的合法性校验。

- 类型

客户端类型用于用于返回当前轻客户端使用的共识算法类型定义。

```go
func (cs ClientState) ClientType() string
```

- 轻客户端唯一标识

```go
func (cs ClientState) ChainName() string
```

- 最新高度

返回轻客户端当前的最新高度。

```go
func (cs ClientState) GetLatestHeight() Height
```

- 合法性校验

对于当前轻客户端数据的合法性校验

```go
func (cs ClientState) Validate() error
```

- Proof校验规则

- 轻客户端状态

返回当前轻客户端状态。

```go
func (cs ClientState) Status() Status
```

- 延迟确认时间周期

返回当前轻客户端的延迟确认时间周期。

```go
func (cs ClientState) DelayTime() uint64
```

- 延迟确认区块周期

返回当前轻客户端的延迟确认区块周期，例如比特币需要6个区块以上的确认周期。

```go
func (cs ClientState) DelayBlock() uint64
```

- MerklePath 前缀

当前轻客户端的MerklePath前缀。定义在`tics-023`中。

```go
func (cs ClientState) Prefix() Prefix
```

- 初始化轻客户端

```go
func (cs ClientState) Initialize(consensusState consensusState) error
```

- 验证并更新轻客户端状态

```go
func (cs ClientState) CheckHeaderAndUpdateState(header Header) (ClientState, ConsensusState, error)
```

- 验证跨链数据包

利用轻客户端共识状态，以及proof等信息对接收到的跨链数据包进行校验。

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

具体参数解释如下：

- `height`：当前跨链数据包proof所在的高度。
- `prefix`：跨链数据包存储store的名称(storeKey)。
- `proof`： 跨链数据包的merkle证明。
- `sourceChain`：数据包的发送链。
- `destChain`： 数据包的接受链。
- `sequence`： 跨链数据包的顺序。
- `commitmentBytes`： 跨链数据包的承诺(跨链数据包按照相同规则hash运算)。

- 验证跨链数据包Ack

利用轻客户端共识状态，以及proof等信息对跨链数据包确认消息进行校验。

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

具体参数解释如下：

- `height`：当前跨链数据包proof所在的高度。
- `prefix`：跨链数据包存储store的名称(storeKey)。
- `proof`： 跨链数据包的merkle证明。
- `sourceChain`：数据包的发送链。
- `destChain`： 数据包的接受链。
- `sequence`： 跨链数据包的顺序。
- `acknowledgement`： 跨链数据包Ack的承诺(跨链数据包确认消息按照相同规则hash运算)。

- 验证跨链数据包（轻客户端状态清理）

当需要清理轻客户端过期的状态信息时，可以发送清理轻客户端跨链数据包来清理状态。

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

具体参数解释如下：

- `height`：当前跨链数据包proof所在的高度。
- `prefix`：跨链数据包存储store的名称(storeKey)。
- `proof`： 跨链数据包的merkle证明。
- `sourceChain`：数据包的发送链。
- `destChain`： 数据包的接受链。
- `sequence`： 跨链数据包的顺序。
- `cleanCommitmentBytes`： 清理轻客户端跨链数据包的承诺(跨链数据包确认消息按照相同规则hash运算)。

- 共识状态

共识状态定义了轻客户端在某个高度的共识结果，一般以`merkle root`的形式定义，这个里的`merkle root`可能是存储根，也可能是交易根，具体实现由轻客户端定义。

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

## 交易

### 更新轻客户端状态

用户提交跨链数据包的时候，必须先更新轻客户端状态，然后再提交交易。如果源链非即时最终性，需要延后区块确认交易，那么用户在提交跨链数据包前N个区块更新轻客户端状态。例如源链需要延迟6个区块确认，用户提交的跨链数据包proof高度必须满足：

```text
 proofHeight - clientState.GetLatestHeight() >= 6
```

更新轻客户端的交易结构定义为：

```go
type MsgUpdateClient struct {
    ChainName string
    Header Header
}
```

更新过程示例如下：

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

## 轻客户端管理

TIBC的轻客户端在hub上采用Gov的方式进行管理，在其他链（例如以太坊）可以采用其他方式，比如多签。

### 创建轻客户端提议

```go
type CreateClientProposal struct {
    ChainName      string
    ClientState    ClientState
    ConsensusState ConsensusState
}
```

如果在以太坊上，`ChainName`指以太坊上部署的某个轻客户端的合约地址，利用`CreateClientProposal`实现向管理合约注册并初始化的该轻客户端。注意，轻客户端的写入方法必须由管理合约调用，其他账户不能修改合约状态，以保证轻客户端的安全性。简要流程如下：

```text
 部署管理合约  -> 部署轻客户端合约 -> 向管理合约注册轻客户端 -> 轻客户端合约状态更新
```

示例如下：

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

### 升级轻客户端提议

当轻客户端过期或者轻客户端状态不正确时，可以使用升级的方式强制更新轻客户端状态。

```go
type UpgradeClientProposal struct {
    ChainName      string
    ClientState    ClientState
    ConsensusState ConsensusState
}
```

过程示例如下：

```go
func UpgradeClient(ctx sdk.Context, chainName string, upgradedClient ClientState, upgradedConsState ConsensusState){
    AssertClientExist(chainName)
    SetClientState(chainName, upgradedClient)
    SetConsensusState(chainName,updatedClientState.GetLatestHeight() newConsensusState)
}
```

### 中继器注册提议

为保证跨链数据包及时、完整的中继到目标链，需要部署中继器程序，扫描源链上的交易，并将其中继到目标链。中继器需要完成的工作：

- 更新来源链在目标链上轻客户端状态。
- 中继跨链交易到目标链。

考虑到跨链数据的安全性，中继器采用投票注册的方式，中继链需要需要对数据的来源进行身份认证，只有认证通过的中继器发送的交易才能被中继链接受。注册提议设计如下：

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
