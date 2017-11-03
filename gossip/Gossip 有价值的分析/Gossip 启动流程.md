# 整个文档是从上至下的包含关系

## 1
peer 初始化 Gossip 提供了哪些参数？未提供哪些？ => peer 在给全局变量 `gossipServiceInstance = &gossipServiceImpl` 时 提供了 mcs/ peerIdentity/ secAdv，deliveryFactory 提供了一个空结构体，privateHandlers/ leaderelection/ deliveryService 只是初始化了空值；gossip 字段是利用 integration.NewGossipComponent(...) 将给定的 grpc server 与新的 Gossip Componet 绑定

### 2
`gossip, err := integration.NewGossipComponent(...)` （**gossip/integration/integration.go 文件下**）这一步从配置文件中读取了很多配置，并放置在 `type gossipServiceImpl struct`（**gossip/gossip/gossip_impl.go 文件下**）下的 config 字段中，接着调用 gossip.NewGossipService(....)

#### 3
`func NewGossipService(...) Gossip {...}`（**gossip/gossip/gossip_impl.go 文件中**）该函数配置　gossipinstance（同文件下的 gossipServiceImpl 结构体），然后开启了两个 goroutine
```go
go g.start()
go g.connect2Bootstrappeers()
```
---
# Ｇossip 服务启动与初始化的详细分析 
Ｇossip 服务最初通过 `/peer/node/start.go` 下对 `service.InitGossipService()` 函数的调用来初始化 Gossip 服务，经过一系列的函数调用，找到 Gossip 服务最重要的部分在于 `gossip/gossip/gossip_impl.go`　文件下的 `func NewGossipService　(conf *Config, s *grpc.Server, secAdvisor api.SecurityAdvisor, mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType, secureDialOpts api.PeerSecureDialOpts) Gossip`　函数的使用，该函数返回了一个与一 gRPC server 相关联的 gossip instance，我们重点分析这个函数。

在 `func NewGossipService(...) Gossip`函数下，在对 `gossipServiceImpl` 实例初始化的过程中，开启了许多 goroutine 来启动相关服务，下文主要是分析开启的各个 goroutine。

### 1.　`go store.expirationRoutine()` 的启动
在配置 gossipServiceImpl 的 stateInfoMsgStore 字段时，？？？还没看懂

- `g.stateInfoMsgStore = g.newStateInfoMsgStore()`
	- `msgstore.NewMessageStoreExpirable(...)`
		- `go store.expirationRoutine()`

### 2. `go idMapper.periodicalPurgeUnusedIdentities()` 的启动
？？？没看懂

### 3. `go p.periodicEmit()`　的启动
该线程执行 emit

### 4. `g.disc = discovery.NewDiscoveryService(...)`　中启动的多个 goroutine

```go
go d.periodicalSendAlive()
go d.periodicalCheckAlive()
go d.handleMessages()
go d.periodicalReconnectToDead()
go d.handlePresumedDeadPeers()
```



