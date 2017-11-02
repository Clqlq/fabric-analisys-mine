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

#### 4
##### 4.1 `func (g *gossipServiceImpl) start()`



##### 4.2 `func (g *gossipServiceImpl) connect2Bootstrappeers()`
- 读取 gossipServiceImpl 中 config 字段中的 BootstrapPeers 字段：该字段代表 core.yaml 中的 peer.gossip.bootstrap 配置。该配置表示 peer 在 startup 时回去连接的其他 peers（PS：所给的 endpoints 必须是同一个 Org　下的 peers 的 endpoint）（PS：当前配置中只给了 127.0.0.1.7051）

