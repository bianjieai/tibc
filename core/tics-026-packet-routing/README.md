
| tics | title          | stage | category | kind      | requires | author           | created    | modified |
| ---- | -------------- | ----- | -------- | --------- | -------- | ---------------- | ---------- | -------- |
| 26   | Packet Routing | draft | TIBC/TAO | interface | 2,5,20  | dgsbl@bianjie.ai | 2021-07-26 |          |

## 概要

路由模块是辅助模块的默认实现，它将接受外部数据报并调用TIBC协议处理程序来处理数据包中继。路由模块保留一个模块查找表，当接收到数据包时，它可以使用该表查找和调用模块，因此外部relayer只需要将数据包中继到路由模块。中继链上的路由模块需要维护一个路由白名单，对packet做路由控制。

### 动机

默认的TIBC处理程序使用接收器调用模式，其中模块必须单独调用TIBC处理程序，以便绑定到端口、发送和接收数据包，这是灵活和简单的，但理解起来有点棘手，可能需要中继进程的额外工作，relayer进程必须跟踪许多模块的状态。本标准描述了一个TIBC“路由模块”，用于自动化最常见的功能、路由数据包并简化relayer的任务。

在多跳跨链环境中，relay chain中的路由模块需要维护一个路由白名单，对跨链数据包进行路由控制。

### 所需属性

- 当模块需要对数据包执行操作时，路由模块应该回调模块上指定的处理程序函数。
- relay chain通过路由模块对packet进行路由控制。通过白名单对相应数据报拦截或放行。

### 模块回调接口

模块必须向路由模块公开以下功能签名，这些签名在收到各种数据报后即被调用：

```typescript
function onRecvPacket(packet: Packet): bytes {
  // defined by the module, returns acknowledgement
}

function onAcknowledgePacket(packet: Packet) {
  // defined by the module
}

function onCleanPacket(packet: Packet) {
  // defined by the module
}

function validatePacket(packet: Packet) error {
  // defined by the module
}
```

必须抛出异常以指示失败，拒绝传入的数据包等。

它们组合在一个`TIBCModule`接口中：

```typescript
interface TIBCModule {
  onRecvPacket: onRecvPacket,
  onAcknowledgePacket: onAcknowledgePacket,
  onCleanPacket: onCleanPacket,
  validatePacket: validatePacket
}
```

实现了TIBCModule的模块对象存储在`Router`中统一管理，router对象在初始化app时就写入内存中

```go
// The router is a map from module name to the TIBCModule
type Router struct {
   router: map[string]TIBCModule
   sealed: boolean
}
```

`Router`提供以下方法：

```go
// Seal prevents the Router from any subsequent route handlers to be registered.
// Seal will panic if called more than once.
func Seal(){
	if router.sealed {
		panic("router already sealed")
	}
	router.sealed = true
}

// Sealed returns a boolean signifying if the Router is sealed or not.
func Sealed() boolean {
  return router
}

// AddRoute adds TIBCModule for a given module name. It returns the Router
// so AddRoute calls can be linked. It will panic if the Router is sealed.
func AddRoute(module string, cbs TIBCModule) *Router {
	if router.sealed {
		panic(fmt.Sprintf("router sealed; cannot register %s route callbacks", module))
	}
	if router.HasRoute(module) {
		panic(fmt.Sprintf("route %s has already been registered", module))
	}
                                                          
	router.routes[module] = cbs
	return router
}

// UpdateRoute updates TIBCModule for a given module name. It returns the Router
// so UpdateRoute calls can be linked. It will panic if the Router is sealed.
func UpdateRoute(module string, cbs TIBCModule) *Router {
	if router.sealed {
		panic(fmt.Sprintf("router sealed; cannot register %s route callbacks", module))
	}
	if !router.HasRoute(module) {
		panic(fmt.Sprintf("route %s has not been registered", module))
	}
                                                          
	router.routes[module] = cbs
	return router
}

// HasRoute returns true if the Router has a module registered or false otherwise.
func HasRoute(module string) boolean {
	_, ok := router.routes[module]
	return ok
}

// GetRoute returns a TIBCModule for a given module.
func GetRoute(module string) (TIBCModule, boolean) { {
	if !router.HasRoute(module) {
		return nil, false
	}
	return router.routes[module], true
}
```



### Routing Rules

路由模块还维护一个Routing Rules作为路由白名单，routing Rules所需属性为：

- Hub 上实现。
- 由管理模块或 Gov 模块配置。
- 配置白名单条目格式为src,dest,port，src为起始链chain ID，dest为目标链chain ID，port为模块绑定端口。例：irishub, wenchangchain, nft。
- 支持通配符，例：irishub, wenchangchain, *。
- 规则有重叠时取并集。
- 仅对 Packet 有效，不拦截 ACK。



所有的rule以string数组传入，拼成一条字符串进行持久化。rules放在RoutingRuleManager中进行管理。RoutingRuleManager结构如下。

```go
type RoutingRuleManager struct {
  RoutingRules string 
}
```

RoutingRuleManager需提供以下方法:

```go
func (rrm RoutingRuleManager) SetRules(rules []string) (error) {
  for i, rule range rules{
    err := rrm.ValidateRule(rule)
  }
  rrm.RoutingRules, err := CombineRulesString(rules)
  return err
}

func (rrm RoutingRuleManager) GetRules() (rules []string){
  rules = SplitRulesString(rrm.Routing)
  return rules
}

// ValidateRule performs rule string validation returning an error
func (rrm RoutingRuleManager) ValidateRule(rule string) error {
}
```








### 属性和不变式

- 代理端口绑定是先到先得服务：一旦模块通过TIBC路由模块绑定到端口，只有该模块才能使用该端口，直到模块释放它。

## 向后兼容

不适用。


## 向前兼容

路由模块与TIBC处理程序接口紧密相连。

## 示例实现

即将推出。

