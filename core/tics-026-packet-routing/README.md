
| tics | title          | stage | category | kind      | requires | author           | created    | modified |
| ---- | -------------- | ----- | -------- | --------- | -------- | ---------------- | ---------- | -------- |
| 26   | Packet Routing | draft | TIBC/TAO | interface |  2,5,20  | dgsbl@bianjie.ai | 2021-07-26 |          |

## Synopsis

The routing module is a default implementation of a secondary module which will accept external datagrams and call the TIBC handler to deal with data packet relay. The routing module keeps a lookup table of modules, which it can use to look up and call a module when a packet is received, so that external relayers need only to relay packets to the routing module. The routing module on the relay chain needs to maintain a routing whitelist to control the packet routing.

### Motivation

The default TIBC handler uses a receiver call pattern, where modules must individually call the TIBC handler in order to bind to ports, send and receive packets, etc. This is flexible and simple but is a little bit tricky to understand and may require extra work on the part of relayer processes, which must track the state of many modules. This standard describes an TIBC "routing module" to automate most common functionality, route packets, and simplify the task of relayers.

In a multi-hop cross-chain environment, the routing module on the relay chain needs to maintain a routing whitelist to control the routing of cross-chain data packets.

### Desired Properties

- Routing modules should call back the specified handler function on the module when packets need to be acted upon.
- The relay chain should be able to control the packet routing through routing modules and to intercept or release corresponding datagram according to the whitelist.

### Module callback interface

Modules must expose the following function signatures to the routing module, which are called upon the receipt of various datagrams:

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

Exceptions must be thrown to indicate failure, reject the incoming packet, etc.
 
These are combined in a `TIBCModule` interface: 

```typescript
interface TIBCModule {
  onRecvPacket: onRecvPacket,
  onAcknowledgePacket: onAcknowledgePacket,
  onCleanPacket: onCleanPacket,
  validatePacket: validatePacket
}
```

The module objects which implemented TIBC module are stored in `Router` for unified management, and the router object is written into the memory when the app is initialized.

```go
// The router is a map from module name to the TIBCModule
type Router struct {
   router: map[string]TIBCModule
   sealed: boolean
}
```

`Router` provides the following approaches:

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

Routing modules also maintain routing rules as the routing whitelist, the desired properties of which are:

- Implementation on the Hub.
- Routing rules should be configured by the management module or the Gov module.
- Configuration of whitelist entry formats should be src, dest, port, where the src is the source chain ID, dest is the destination chain ID, and port is the port bonded by a module.
- Support wildcard characters, for example, irishub, wenchangchain, *.
- A union of rules should be adopted where rules overlap.
- Routing rules should only be valid for packet and not intercept ACK.

All rules are passed through strings to merge into a single persistent string. Rules are stored under `RoutingRuleManager` for management, which is as follows:

```go
type RoutingRuleManager struct {
  RoutingRules string 
}
```

`RoutingRuleManager` should provide the following approaches:

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

### Properties & Invariants

- Proxy port binding is first-come-first-serve: once a module binds to a port through the TIBC routing module, only that module can utilize that port until the module releases it.

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

Routing modules are closely tied to the TIBC handler interface.

## Example Implementation

Coming soon.

