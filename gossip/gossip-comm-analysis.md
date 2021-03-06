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

- `func (c *commImpl) SendWithAck(msg *proto.SignedGossipMessage, timeout time.Duration, minAck int, peers ...*RemotePeer) AggregatedSendResult` SendWithAck 向远程 peers 发送 message，等待最小数量的 Ack （minAck）或者只到计时器过期
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

### connectionStore
- `func (cs *connectionStore) getConnection(peer *RemotePeer) (*connection, error)`　新建了一个 connection，得到 *connection 并将该 *connection　存储进 connectionStore　的 pki2Conn 字段这个字典中。在返回之前，创建了一个 goroutine　调用 *connnection.serviceConnection　方法

- `func (cs *connectionStore) connNum() int`　就是返回 *connectionStore.pki2Conn 的键对值数目

- `func (cs *connectionStore) closeConn(peer *RemotePeer)` 先锁住该 cs *connectionStore，通过删除 pki to connection 的键对值来关闭一个连接

- `func (cs *connectionStore) shutdown()` 就是关闭所有的连接

- `func (cs *connectionStore) onConnected(serverStream proto.Gossip_GossipStreamServer, connInfo *proto.ConnectionInfo) *connection`　注册新的连接到 connectionStore　中，如果之前 connection 就存在的话，先调用 *connection.close()　来关闭该连接

- `func (cs *connectionStore) registerConn(connInfo *proto.ConnectionInfo, serverStream proto.Gossip_GossipStreamServer) *connection` 新建一个 connection 并存入 *connectionStore.pki2Conn 字典中

- `func (cs *connectionStore) closeByPKIid(pkiID common.PKIidType) ` 从 *connection.pki2Conn 字典中删除对应项

### connection
- `func (conn *connection) close()` 
	- 首先调用 `*connection.drainOutputBuffer` 把 `*connection.outBuff`　清空
	- 然后关闭 `*connection.clientStream`（到远程端点的 client-side stream）
	- 然后关闭 `*connection.conn`（到远程端点的 gRPC 连接）
	- 执行 `*connection.cancel`
	
- `func (conn *connection) send(msg *proto.SignedGossipMessage, onErr func(error), shouldBlock blockingBehavior)`　
	- 先锁定该 conneciton
	- 将要发送的消息打包成 msgSending　结构体的形式
	- 将该结构体的指针传入 *connection.outBuff 通道中

- `func (conn *connection) serviceConnection() error` 
	- 新建了两个通道 errChan 与 msgChan 分别接受 error 信息与 *proto.SignedGossipMessage　类型的信息， errChan 大小为1，msgChan 大小为20
	- 启动一个 goroutine 调用 conn.readFromStream(errChan, msgChan) （感觉就是通过 stream.Recv() 获取 envelope，然后将里面的 message　写入通道中，所以方法名叫 readFromStream）
	- 启动一个 goroutine 调用 conn.writeToStream()
	- select 语句，把相关信息写入对应通道

- `func (conn *connection) writeToStream()` 没太明白

- `func (conn *connection) drainOutputBuffer()`　清空 *connection.outBuff
	- 通过不断执行`<- *connection.outBuff`即可将通道中的信息一个个释放
	
- `func (conn *connection) readFromStream(errChan chan error, msgChan chan *proto.SignedGossipMessage)`　感觉就是通过 stream.Recv() 获取 envelope，然后将里面的 message　写入通道 msgChan 中


- `func (conn *connection) getStream() stream`　
	- 先判断 *connection.clientStream 与 *connection.serverStream 如果都不为空则肯定哪儿有问题，返回 error
	- 先尝试返回 *connection.clientStream　
	- 若前者为空则尝试返回 *connection.serverStream 
	- 若前两者都为空则返回 nil

## crypto.go
- `func writeFile(filename string, keyType string, data []byte) error` 先创建文件再调用 `pem.Encode()`　方法去写文件

- `func GenerateCertificatesOrPanic() tls.Certificate`　生成证书
	- 执行 `privateKey, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)`　生成一对公钥与私钥
	- 调用 `x509.CreateCertificate()` 来基于所给模板（包括上面得到的 privateKey.PublicKey 与 privateKey，生成新证书的 rawBytes
	- 调用上面提到的 writeFile 函数，将 rawBytes 写入证书文件 certKeyFile 中去
	- 执行 `privBytes, err := x509.MarshalECPrivateKey(privateKey)`　得到私钥的 ASN.1, DER 格式的二进制数据 privBytes
	- 同样调用上面提到的 writeFile 函数，将 privBytes 写入证书文件 privKeyFile 中去
	- 执行 `cert, err := tls.LoadX509KeyPair(certKeyFile, privKeyFile)`　得到证书 cert Certificate（Certificate　结构体是一个或多个证书的链）
	- 返回 cert
	- 删除 certKetFile 与 privKeyFile 两个文件

- `func certHashFromRawCert(rawCert []byte) []byte`　对给定的二进制数据进行 SHA2-256　hash 运算，返回该 hash　值

- `func extractCertificateHashFromContext(ctx context.Context) []byte`　从流中提取证书的 hash


## demux.go
- ChannelDeMultiplexer 是一个可以接受通道 registrations (AddCahnnel) 与 publications (DeMultiplex) 的结构体，并且可以根据 predicate　将 publications　广播到 registrations
```go
type ChannelDeMultiplexer struct {
	channels []*channel
	lock     *sync.RWMutex
	closed   int32
}
```
- channel 的定义
```go
type channel struct {
	pred common.MessageAcceptor	//MessageAcceptor1是一个 predicate，用于确定创建　MessageAcceptor　实例的订阅者对哪些 messages 感兴趣
	ch   chan interface{}
}
```
- `func NewChannelDemultiplexer() *ChannelDeMultiplexer `创建一个新的 ChannelDeMultiplexer

- `func (m *ChannelDeMultiplexer) Close() `　关闭通道

- `func (m *ChannelDeMultiplexer) AddChannel(predicate common.MessageAcceptor) chan interface{}` 给定特定的 predicate　注册一个通道

- `func (m *ChannelDeMultiplexer) DeMultiplex(msg interface{})`　将消息广播到由 AddChannel 调用返回并保存对应 predicate　的所有通道

## msg.go
- ReceivedMessageImpl 是 ReceivedMessage 的一个实现
```go
type ReceivedMessageImpl struct {
	*proto.SignedGossipMessage
	lock     sync.Locker
	conn     *connection
	connInfo *proto.ConnectionInfo
}
```
- `func (m *ReceivedMessageImpl) GetSourceEnvelope() *proto.Envelope` 返回构造该 ReceivedMessage　的 Envelope

- `func (m *ReceivedMessageImpl) Respond(msg *proto.GossipMessage)`　发送一个 msg 到发送该 ReceivedMessageImpl　的源去

- `func (m *ReceivedMessageImpl) GetGossipMessage() *proto.SignedGossipMessage` 返回内部的 GossipMessage

- `func (m *ReceivedMessageImpl) GetConnectionInfo() *proto.ConnectionInfo` 返回关于发送该 message　的远程 peer　的信息

- `func (m *ReceivedMessageImpl) Ack(err error)`为该 message　发送 acknowledgement










