## 代码结构
```
/discovery
|	|- discovery.go
|	|- discovery_impl.go
```
---


## discovery.go

```go
// CryptoService is an interface that the discovery expects to be implemented and passed on creation
type CryptoService interface {
	// ValidateAliveMsg 验证 Alive message
	ValidateAliveMsg(message *proto.SignedGossipMessage) bool

	// SignMessage 对一个 messae 进行签名
	SignMessage(m *proto.GossipMessage, internalEndpoint string) *proto.Envelope
}

// EnvelopeFilter 会也可能不会删除给定 SignedGossipMessage 来源的部分 Envelope
type EnvelopeFilter func(message *proto.SignedGossipMessage) *proto.Envelope

/ Sieve 基于一些准则（criteria）定义了能够被准许发送往一些远程 peer
// 的 messages
// Sieve 返回对是否准予发送一个给定的 message 的判断
type Sieve func(message *proto.SignedGossipMessage) bool

// DisclosurePolicy定义给定远程 peer 有资格知道哪些消息
// 以及在给定的 SignedGossipMessage 中有资格知道哪些消息
// 1) 一个 Sieve 针对一个给定的远程 peer
//	Sieve 应用在所讨论的每个 peer
//	并且输出一个 message 是否应该被披露给远程 peer
// 2) 一个 EnvelopeFilter 针对一个给定 SignedGossipMessage
//	EnvelopeFilter 可能会删除 SignedGossipMessage　源于的 Envelop　的部分
type DisclosurePolicy func(remotePeer *NetworkMember) (Sieve, EnvelopeFilter)

// CommService 是期望被实现的接口
type CommService interface {
	//　Gossip gossip 一个 message
	Gossip(msg *proto.SignedGossipMessage)

	// SendToPeer 发送一个 message　到给定的 peer
	// The nonce can be anything since the communication module handles the nonce itself
	SendToPeer(peer *NetworkMember, msg *proto.SignedGossipMessage)

	// Ping 检测一个远程 peer　是否还有回复
	Ping(peer *NetworkMember) bool

	// Accept 为发送自远程 peers 的成员 messages　返回一个只读的通道
	Accept() <-chan *proto.SignedGossipMessage

	//　PresumedDead 为那些被假定已经死亡的 peers　返回一个只读通道
	PresumedDead() <-chan common.PKIidType

	// CloseConn orders to close the connection with a certain peer
	//　CloseConn 命令关闭与一个特定 peer 的连接
	CloseConn(peer *NetworkMember)
}

// NetworkMember 是一个 peer 的代表
type NetworkMember struct {
	Endpoint         string
	Metadata         []byte
	PKIid            common.PKIidType
	InternalEndpoint string
	Properties       *proto.Properties
}

// String 返回 NetworkMember 的字符串表示
func (n *NetworkMember) String() string {
	return fmt.Sprintf("Endpoint: %s, InternalEndpoint: %s, PKI-ID: %v, Metadata: %v", n.Endpoint, n.InternalEndpoint, n.PKIid, n.Metadata)
}

// PreferredEndpoint 计算去连接的endpoint
// 相比标准的 endpoint，更偏爱内部 endpoint 
func (n NetworkMember) PreferredEndpoint() string {
	if n.InternalEndpoint != "" {
		return n.InternalEndpoint
	}
	return n.Endpoint
}

// PeerIdentification 包括了一个远程 peer 的 PKI-ID
// 与对 peer 是否属于当前 org 判断
type PeerIdentification struct {
	ID      common.PKIidType
	SelfOrg bool
}

type identifier func() (*PeerIdentification, error)

// Discovery 是代表 discovery 模块的接口
type Discovery interface {

	// Lookup 返回一个 NerworkMember，找不到则返回 nil
	Lookup(PKIID common.PKIidType) *NetworkMember

	// Self 返回实例的成员信息
	Self() NetworkMember

	// UpdateMetadata 更新实例的 metadata
	UpdateMetadata([]byte)

	// UpdateEndpoint 更新实例的 endpoint
	UpdateEndpoint(string)

	// Stop 停止该实例
	Stop()

	// GetMembership 返回在视图中还存活的成员
	GetMembership() []NetworkMember

	// InitiateSync 使得实例去询问给定数量的 peer 的成员信息
	InitiateSync(peerNum int)

	// Connet 使得这个实例去连接一个远程实例
	// identifier 参数是一个函数，被用来验证 peer 的身份
	// 获取它的 PKI-ID，对它是否属于当前 peer 的组织的判断
	// 以及对该操作是否成功的判断
	Connect(member NetworkMember, id identifier)
}
```
---

## discovery_impl.go

- gossipDiscoveryImpl 结构体
```go
type gossipDiscoveryImpl struct {
	incTime         uint64
	seqNum          uint64
	self            NetworkMember
	deadLastTS      map[string]*timestamp     // H
	aliveLastTS     map[string]*timestamp     // V
	id2Member       map[string]*NetworkMember // all known members
	aliveMembership *util.MembershipStore
	deadMembership  *util.MembershipStore

	msgStore *aliveMsgStore

	comm  CommService
	crypt CryptoService
	lock  *sync.RWMutex

	toDieChan        chan struct{}
	toDieFlag        int32
	port             int
	logger           *logging.Logger
	disclosurePolicy DisclosurePolicy
	pubsub           *util.PubSub
}
```

- `func NewDiscoveryService(self NetworkMember, comm CommService, crypt CryptoService, disPol DisclosurePolicy) Discovery` 新建并返回了一个 discovery service 的实例。在做了初始化与简单的自我配置检查后，还开启了几个 goroutine，可以从方法名中窥得一二
```go
	go d.periodicalSendAlive()
	go d.periodicalCheckAlive()
	go d.handleMessages()
	go d.periodicalReconnectToDead()
	go d.handlePresumedDeadPeers()

```
---
本文件下有几个对外公开的方法： Lookup / Connect / Updatemetadata / UpdateEndpoint / Self / Stop /	Add / Checkvalid / InitiateSync

- `func (d *gossipDiscoveryImpl) Lookup(PKIID common.PKIidType) *NetworkMember` 首先判断所给 PKIID 是否代表自己，如果代表自己则返回 self，否则在 d.id2Member 中查询返回（不存在会返回 nil）

- `func (d *gossipDiscoveryImpl) Connect(member NetworkMember, id identifier)` 连接远程 peer
	- 首先判断所给 member 的 InternalEndpoint 与 Endpoint，如果是实例自己，则停止连接，否则继续
	- 启动一个 goroutine 运行一匿名函数，函数中启动一 for 循环，尝试连接 peer，最到尝试次数为 maxConnectionAttempts=120
		- 调用函数 id() 获取 id
		- 调用 d.createMembershipRequest 方法生成新的 membership request
		- 对生成的 membership request 进行签名
		- 随机生成一个 NONCE 并添加到签名过后的 message 上
		- 调用 d.sendUntilAcked 启动一个 goroutine  把请求发往 peer

- `func (d *gossipDiscoveryImpl) UpdateMetadata(md []byte)`　简单的更新 d.self.Metadata　而已（需要先加锁，更新，再解锁）

- `func (d *gossipDiscoveryImpl) UpdateEndpoint(endpoint string)` 简单的更新 d.self.Endpoint 而已（需要先加锁，更新，再解锁）

- `func (d *gossipDiscoveryImpl) Self() NetworkMember`　返回自己的 NetworkMember 信息

- `func (d *gossipDiscoveryImpl) Stop()` 停止该实例（具体实现还没看懂）

- `func (d *gossipDiscoveryImpl) InitiateSync(peerNum int)` 从 d.aliveMembership　中选择 peerNum 个 peer 初始化连接

- `func (d *gossipDiscoveryImpl) GetMembership() []NetworkMember`　提取 d.aliveMembership　中所有 peer　的信息，组合成 NetworkMember 切片返回

---
```go
type aliveMsgStore struct {
	msgstore.MessageStore
}
```

- `func (s *aliveMsgStore) Add(msg interface{}) bool` 将 msg 添加到 s.MessageStore 中

- `func (s *aliveMsgStore) CheckValid(msg interface{}) bool` 






























