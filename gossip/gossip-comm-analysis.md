##  代码结构
```
/comm
|	|- ack.go
|	|- comm.go
|	|- comm_impl.go
|	|- conn.go
|	|- crypto.go
|	|- demux.go
|	|- msg.go
```
---

ack： 将签名后的 gossipmessage发送给　peer 并获取　ack 

comm：接口中定义了很多方法，比如发送消息，探测存活，握手；一些结构体

comm_impl：将 comm　中定义的方法实现了（我觉得这部分很重要）	
			

---

## comm.go
主要是定义了 Comm 接口，　**comm_impl.go** 实现了这些接口

```go

// Comm 是一个可以和其他嵌入了 CommModule的 peers　交流的对象
type Comm interface {

	//　GetPKIid　返回该实例的 PKI id
	GetPKIid() common.PKIidType

	// Send 发送了一个 message　到远程 peers
	Send(msg *proto.SignedGossipMessage, peers ...*RemotePeer)

	// SendWithAck 发送一个 message　到远程 peers，等待一个最小数目的　ack，或者知道 timeout 到期
	SendWithAck(msg *proto.SignedGossipMessage, timeout time.Duration, minAck int, peers ...*RemotePeer) AggregatedSendResult

	// Probe 探测远程 peers　是否存活，如果该 peer　回复则返回 nil，否则返回 error
	Probe(peer *RemotePeer) error

	// Handshake 验证远程 peer
	// 如果成功则返回 its identity, nil
	// 失败则返回 nil, error
	Handshake(peer *RemotePeer) (api.PeerIdentityType, error)

	// Accept 为与某个谓词匹配的其他节点发送的消息返回专用的只读通道
	// 每个来自该通道的消息都可用于将回复发送回发送方
	Accept(common.MessageAcceptor) <-chan proto.ReceivedMessage

	// PresumedDead 为那些疑似下线了的节点终端返回一个只读的通道
	PresumedDead() <-chan common.PKIidType

	// CloseConn　关闭通往某一确定终端的连接
	CloseConn(peer *RemotePeer)

	// Stop 停止该 module
	Stop()

```
---

## comm_impl.go


## 以下是草稿

我们根据 comm.go 接口中的方法来看一下在 comm_impl.go 中的具体实现。

comm_impl.go 中定义了一个名为 commImpl 的结构体，该文件下主要是实现该结构体类型的各种方法

```go
type commImpl struct {
	pubSub         *util.PubSub
	selfCertHash   []byte
	peerIdentity   api.PeerIdentityType
	idMapper       identity.Mapper
	logger         *logging.Logger
	opts           []grpc.DialOption
	secureDialOpts func() []grpc.DialOption
	connStore      *connectionStore
	PKIID          []byte
	deadEndpoints  chan common.PKIidType
	msgPublisher   *ChannelDeMultiplexer
	lock           *sync.Mutex
	lsnr           net.Listener
	gSrv           *grpc.Server
	exitChan       chan struct{}
	stopWG         sync.WaitGroup
	subscriptions  []chan proto.ReceivedMessage
	port           int
	stopping       int32
}
```

- `func (c *commImpl) GetPKIid() common.PKIidTypeP`没什么好看的

- `func (c *commImpl) Send(msg *proto.SignedGossipMessage, peers ...*RemotePeer) `
	- 执行一个 for 循环，调用 `func (c *commImpl) sendToEndpoint(peer *RemotePeer, msg *proto.SignedGossipMessage, shouldBlock blockingBehavior)` 将 msg *proto.SignedGossipMessage 发送给 peers 中的每一个 peer
	
- `func (c *commImpl) sendToEndpoint(peer *RemotePeer, msg *proto.SignedGossipMessage, shouldBlock blockingBehavior)`
	- 执行 conn, err := c.connStore.getConnection(peer) 获得一个 connection 结构体的指针
	- 执行 conn.send(msg, disConnectOnErr, shouldBlock) 调用 conn 的 send 方法将 msg 发送出去（send 方法的具体实现要看 conn.go 文件下的源码，不过大致看了一眼，貌似最后是把一个信息发送到了某一个通道）
	- 执行 c.disconnect(peer.PKIID) 断开连接（disconnect 方法做了两件事，一个是把 pkiID 发送到了 `*connection.deadEndpoints` 通道中；另一件是把 `*connection.pki2Conn` 中 pkiID 的键对值删除）

- `func (c *commImpl) SendWithAck(msg *proto.SignedGossipMessage, timeout time.Duration, minAck int, peers ...*RemotePeer) AggregatedSendResult` SendWithAck 向远程 peers 发送 message，等待最小数量的 Ack （minAck）或者知道计时器过期
	- 对 peers 中的每一个 peer 分别做处理
	- 执行 `ackOperation := newAckSendOperation(sndFunc, waitForAck)`，返回一个 ackSendOperation 结构体指针
	- 执行 `return ackOperation.send(msg, minAck, peers...)`，`*ackOperation.send` 方法大概就是对返回的 ack 做判断最后返回的是一个 []SendResult（该方法中也涉及了一些通道操作没太看明白） 

- `func (c *commImpl) Probe(remotePeer *RemotePeer) error`
	- 主要是调用了 GossipClient 中的 Ping 方法（GossipClient 与 Ping 的详细定义在 message.pb.go 中）
	- Ping 方法是通过调用 grpc.Invoke 来实现的
	- 最后返回 err error	

- `func (c *commImpl) Handshake(remotePeer *RemotePeer) (api.PeerIdentityType, error)` 
	- 调用 `*GossipClient.Ping` 看远程 peer 是否能 ping 通
	- 调用 `*GossipClient.GossipStream`，检查返回的 err （这步拿来干嘛的？）
	- 调用 `*commImpl.authenticateRemotePeer`，返回 `connInfo *gossip.ConnectionInfo`
	- 最后主要是判断 `bytes.Equal(connInfo.ID, remotePeer.PKIID)`，如果两个 ID 不同则返回 error 信息，相同则返回 `connInfo.Identity, nil`

- `func (c *commImpl) Accept(acceptor common.MessageAcceptor) <-chan proto.ReceivedMessage` 返回的是一个只读的通道，容量为10（定义中，通道里存储的是 proto.ReceiedMessage 类型的数据，实际代码中没看懂存入的数据）

- `func (c *commImpl) PresumedDead() <-chan common.PKIidType`返回一个通道 —— `chan common.PKIidType` 类型的通道

- `func (c *commImpl) CloseConn(peer *RemotePeer)` 
	- 执行 `c.connStore.closeConn(peer)` 关闭连接，实际就是把 `c *commImpl` 中 connStore 字段（该字段为 `*connectionStore` 类型）中的 pki2Conn 删除对应的键对值


- `func (c *commImpl) Stop()` 对 `c *commImpl` 中的很多字段执行了诸如 Stop/Close/shutdown 之类的操作（具体见代码）

---

## conn.go
在　comm_impl.go　中定义了 commImpl 结构体，其中有一个字段 conn，其类型为　*connectionStore， connectionStore　类型定义如下：
```go
type connectionStore struct {
	logger           *logging.Logger        // logger
	isClosing        bool                   // 这个 connection store 是否正在关闭
	connFactory      connFactory            // 创建一个到远程 peer　的连接
	sync.RWMutex                            // 同步对共享变量的访问
	pki2Conn         map[string]*connection // mapping between pkiID to connections
	destinationLocks map[string]*sync.Mutex //mapping between pkiIDs and locks,
	// 用于防止并发连接建立到同一个远程端点
}
```
connectionStore 中 pki2Conn 字段的类型为 map[string]*connection，connection　是一结构体类型，定义也在本文件下，定义如下：
```go
type connection struct {
	cancel       context.CancelFunc
	info         *proto.ConnectionInfo
	outBuff      chan *msgSending
	logger       *logging.Logger                 // logger
	pkiID        common.PKIidType                // 远程端点的 pkiID
	handler      handler                         // 函数在消息接收时调用
	conn         *grpc.ClientConn                // gRPC连接到远程端点
	cl           proto.GossipClient              // 远程端点的gRPC存根
	clientStream proto.Gossip_GossipStreamClient // client-side stream to remote endpoint
	serverStream proto.Gossip_GossipStreamServer // server-side stream to remote endpoint
	stopFlag     int32                           // indicates whether this connection is in process of stopping
	stopChan     chan struct{}                   // a method to stop the server-side gRPC call from a different go-routine
	sync.RWMutex                                 // synchronizes access to shared variables
}
```
所以本文件下主要就是对上述两种结构体类型定义了很多方法，我们来一一的看一看。

#### connectionStore
- `func (cs *connectionStore) getConnection(peer *RemotePeer) (*connection, error)`　新建了一个 connection，得到 *connection 并将该 *connection　存储进 connectionStore　的 pki2Conn 字段这个字典中。在返回之前，创建了一个 goroutine　调用 *connnection.serviceConnection　方法


#### connection
- `func (conn *connection) serviceConnection() error` 














