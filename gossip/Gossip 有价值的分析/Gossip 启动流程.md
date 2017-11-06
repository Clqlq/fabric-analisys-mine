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
在配置 gossipServiceImpl 的 stateInfoMsgStore 字段时

- `g.stateInfoMsgStore = g.newStateInfoMsgStore()`
	- `msgstore.NewMessageStoreExpirable(...)`
		- `go store.expirationRoutine()`
		
该 goroutine 定期的读取 `gossipServiceImpl` 的 `stateInfoMsgStore`字段中的 msg，然后读取 msg 的 创建时间，与所设置的 msgTTL 做比较，并把相关 msg 从 []*msg 中清楚出去。msgTTL 对应的是 `g.conf.PublishStateInfoInterval` 即 core.yaml 文件中的 `peer.gossip.publishStateInfoInterval` 配置项。（**???但是我比较不解，就是该配置项的注释说该配置项是 把state info msg push 向 peers 的频率**)

### 2. `go idMapper.periodicalPurgeUnusedIdentities()` 的启动
该 goroutine 会每隔6分钟执行一次检查（`case <-time.After(usageThreshold / 10):` )，对于已经被撤销，或者过期，或者长时间未使用（1个小时）的 peer 的 identity，将他们从 gossipServiceImpl 的 idMapper 字段（该字段维护 pkiID 与 peers identities（certificates）之间的映射）的 `pkiID2Cert` 字段键值对中删除

### 3. `go p.periodicEmit()`　的启动
- 该 goroutine 周期性的执行 emit （发送）操作（emit 有两种触发条件，一种是 gossipServiceImpl 的 emitter 字段中存储的 msg 数量大于配置文件中设定的阈值，一种是定期的计时器到期），每隔 10 ms 发送一次，该配置由 core.yaml 的 `peer.gossip.maxPropagationBurstLatency` 配置项决定。
- 另：在 gossipServiceImpl 初始化其 emitter 字段时，给定了４个配置，其中三个分别为 `peer.gossip.propagateIterations`，｀peer.gossip.maxPropagationBurstSize｀，｀peer.gossip.maxPropagationBurstLatency｀，分别代表每个 msg 被发送的次数／缓存 msg 的最大值／连续两次 push msg 之间的最大时间间隔，默认值分别为１／10 ／10ms，每一次将新的 msg 加入 emitter 对应缓存中，都会判断是否已经达到的最大缓存数，如果达到了则执行 emit 操作。 
- emit 操作实际上是将要发送的 msg 打包成 `msgs2beEmitted`的切片，然或传送给刚才一个回调函数——即前面所说的第4个配置，代码中给定的函数是 `gossip/gossip/gossip_impl.go`下的`func (g *gossipServiceImpl) sendGossipBatch(a []interface{})`
- `func (g *gossipServiceImpl) sendGossipBatch(a []interface{})`继续调用`func (g *gossipServiceImpl) gossipBatch(msgs []*proto.SignedGossipMessage)`，gossipBatch 是真正决定将要将 msg  gossip 到哪些 peers 的方法（**关于 gossipBatch 的分析见其他文档~~现在还没写~~**）

### 4. `g.disc = discovery.NewDiscoveryService(...)`　中启动的多个 goroutine

```go
go d.periodicalSendAlive()
go d.periodicalCheckAlive()
go d.handleMessages()
go d.periodicalReconnectToDead()
go d.handlePresumedDeadPeers()
```



