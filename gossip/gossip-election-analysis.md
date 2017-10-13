##  代码结构
```
/election
|	|- adapter.go
|	|- election.go
```
---

## election.go 
- Gossip leader 选举模块
```
算法性质：
	- peers　通过比较 IDs　打破对称
	- 每一个 peer　要么是 leader　要么是 follower
	   目标是在所有 peers　都有同样的 membership view　下保证只有１个 leader
	- 如果网络分区为两个或两个以上的子网，leaders　的数目就是网络的分区数，
	   但是当网络又归为一个时，最后只能有一个 leader
	- peers　之间靠 gossip leadership proposal 或　declaration messages 交流
	
算法伪代码实现：

// variables:
// 	leaderKnown = false
//
// Invariant:
//	Peer listens for messages from remote peers
//	and whenever it receives a leadership declaration,
//	leaderKnown is set to true
//
// Startup():
// 	wait for membership view to stabilize, or for a leadership declaration is received
//      or the startup timeout expires.
//	goto SteadyState()
//
// SteadyState():
// 	while true:
//		If leaderKnown is false:
// 			LeaderElection()
//		If you are the leader:
//			Broadcast leadership declaration
//			If a leadership declaration was received from
// 			a peer with a lower ID,
//			become a follower
//		Else, you're a follower:
//			If haven't received a leadership declaration within
// 			a time threshold:
//				set leaderKnown to false
//
// LeaderElection():
// 	Gossip leadership proposal message
//	Collect messages from other peers sent within a time period
//	If received a leadership declaration:
//		return
//	Iterate over all proposal messages collected.
// 	If a proposal message from a peer with an ID lower
// 	than yourself was received, return.
//	Else, declare yourself a leader

```

- LeaderElectionAdapter　接口被 leader 选举模块用来发送/接受消息与获取成员信息，定义如下：
```go
type LeaderElectionAdapter interface {
	// Gossip gossips 一个 message 到其他 peers
	Gossip(Msg)

	// Accept 返回一个发送 messages 的通道
	Accept() <-chan Msg

	// CreateProposalMessage
	 (isDeclaration bool) Msg

	// Peers 返回一个包含被认为还存活的 peers 的列表
	Peers() []Peer
}

```


- LeaderElectionService 接口是执行 leader 选举算法的对象，定义如下：
```go
type LeaderElectionService interface {
	// IsLeader 返回该 peer 是或不是 leader
	IsLeader() bool

	// Stop 停止该 LeaderElectionService
	Stop()

	// Yield relinquishes the leadership until a new leader is elected,
	// or a timeout expires
	// Yield 放弃领导知道新的 leader 当选，或者超时到期
	Yield()
}

```
- Peer 接口描述了远程 peer
```go
type peerID []byte

type Peer interface {
	// ID 返回 peer 的 ID
	ID() peerID
}
```

- Msg 接口描述了从远程 peer 发来的一个 message
```go
type Msg interface {
	// SenderID 返回发送该 message 的 peer 的 ID
	SenderID() peerID
	// IsProposal 返回该 message 是否是一个 leadership 提案
	IsProposal() bool
	// IsDeclaration 返回该 messgae 是否是一个 leadership 声明
	IsDeclaration() bool
}
```
- `func NewLeaderElectionService(adapter LeaderElectionAdapter, id string, callback leadershipCallback) LeaderElectionService`　创建并返回一个新的 LeaderElectionService

- leaderElectionSvcImpl 是 LeaderElectionService 的一个实现，意味着 leaderElectionSvcImpl　实现了 LeaderElectionService　下的所有方法
```go
type leaderElectionSvcImpl struct {
	id        peerID
	proposals *util.Set
	sync.Mutex
	stopChan      chan struct{}
	interruptChan chan struct{}
	stopWG        sync.WaitGroup
	isLeader      int32
	toDie         int32
	leaderExists  int32
	yield         int32
	sleeping      bool
	adapter       LeaderElectionAdapter
	logger        *logging.Logger
	callback      leadershipCallback
	yieldTimer    *time.Timer
}
```
下面关于 leaderElectionSvcImpl　的各种方法暂不介绍，主要先记住该接口体是 LeaderElectionService 接口的实现，外部调用也只能直接点用接口中声明了的方法。

---

## adapter.go
- msgImpl 结构体是 election.go　文件中 Msg 接口的实现
```go
type msgImpl struct {
	msg *proto.GossipMessage
}

- func (mi *msgImpl) SenderID() peerID
- func (mi *msgImpl) IsProposal() bool
- func (mi *msgImpl) IsDeclaration() bool 
```

- peerImpl 结构体是 election.go 文件中 Peer 接口的实现
```go
type peerImpl struct {
	member discovery.NetworkMember
}

- func (pi *peerImpl) ID() peerID
```

- gossip 接口（感觉这个接口和 election.go 文件中的 LeaderElectionAdapter 接口很像）
```go
type gossip interface {
	// Peers 返回被认为还存活 NetworkMembers
	Peers() []discovery.NetworkMember

	// Accept 返回一个明确的只读通道，该通道中的内容是由那些匹配一个特定 predicate　的节点发来 messages
	// 如果 passThrough 是 false，这些 messages 由 gossip layer 预先处理
	// 如果 passThrough 是 true， gossip layer 不干预这些 messages
	// 并且这些 messages　能够用于将回复送回发送方
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)

	// Gossip 发送一个 message 到该网络的其他 peers
	Gossip(msg *proto.GossipMessage)
}
```
- `func NewAdapter(gossip gossip, pkiid common.PKIidType, channel common.ChainID) LeaderElectionAdapter` 创造新的 leader election adapter

- adapterImpl 结构体，是对 LeaderElectionAdapter 接口的实现
```go
type adapterImpl struct {
	gossip    gossip
	selfPKIid common.PKIidType

	incTime uint64
	seqNum  uint64

	channel common.ChainID

	logger *logging.Logger

	doneCh   chan struct{}
	stopOnce *sync.Once
}
- func (ai *adapterImpl) Gossip(msg Msg) 
- func (ai *adapterImpl) Accept() <-chan Msg 
- func (ai *adapterImpl) CreateMessage(isDeclaration bool) Msg 
- func (ai *adapterImpl) Peers() []Peer
- func (ai *adapterImpl) Stop()

```
































