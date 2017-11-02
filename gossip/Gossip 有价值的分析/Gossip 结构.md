# 1. 
```go
gossip, err = integration.NewGossipComponent(peerIdentity, endpoint, s, secAdv,
			mcs, secureDialOpts, bootPeers...)

// gossipServiceInstance 是一个全局变量
gossipServiceInstance = &gossipServiceImpl{
			mcs:             mcs,
			gossipSvc:       gossip,
			privateHandlers: make(map[string]privateHandler),
			chains:          make(map[string]state.GossipStateProvider),
			leaderElection:  make(map[string]election.LeaderElectionService),
			deliveryService: make(map[string]deliverclient.DeliverService),
			deliveryFactory: factory,
			peerIdentity:    peerIdentity,
			secAdv:          secAdv,
		}

// gossipServiceImpl 结构体的具体定义
type gossipServiceImpl struct {
	gossipSvc
	privateHandlers map[string]privateHandler
	chains          map[string]state.GossipStateProvider
	leaderElection  map[string]election.LeaderElectionService
	deliveryService map[string]deliverclient.DeliverService
	deliveryFactory DeliveryServiceFactory
	lock            sync.RWMutex
	mcs             api.MessageCryptoService
	peerIdentity    []byte
	secAdv          api.SecurityAdvisor
}
```
---

## 2. gossipServiceImpl 结构体
### 2.1 gossipSvc 字段
定义是 gossip/gossip/gossip.go 下的 Gossip interface，实际实现是 gossip/gossip/gossip_impl.go 下的 gossipServiceImpl struct（PS：两 struct 名字相同，注意区分）

```go
type Gossip interface {
	Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
	
	SendByCriteria(*proto.SignedGossipMessage, SendCriteria) error
	
	Peers() []discovery.NetworkMember
	
	PeersOfChannel(common.ChainID) []discovery.NetworkMember
	
	UpdateMetadata(metadata []byte)
	
	UpdateChannelMetadata(metadata []byte, chainID common.ChainID)
	
	Gossip(msg *proto.GossipMessage)
	
	PeerFilter(channel common.ChainID, messagePredicate api.SubChannelSelectionCriteria) (filter.RoutingFilter, error)
	
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)
	
	JoinChan(joinMsg api.JoinChannelMessage, chainID common.ChainID)
	
	LeaveChan(chainID common.ChainID)
	
	SuspectPeers(s api.PeerSuspector)
	
	Stop()
}
```

### 2.1.1 gossipServiceImpl struct
```go
type gossipServiceImpl struct {
	selfIdentity          api.PeerIdentityType
	includeIdentityPeriod time.Time
	certStore             *certStore
	idMapper              identity.Mapper
	presumedDead          chan common.PKIidType
	disc                  discovery.Discovery
	comm                  comm.Comm
	incTime               time.Time
	selfOrg               api.OrgIdentityType
	*comm.ChannelDeMultiplexer
	logger            *logging.Logger
	stopSignal        *sync.WaitGroup
	conf              *Config
	toDieChan         chan struct{}
	stopFlag          int32
	emitter           batchingEmitter
	discAdapter       *discoveryAdapter
	secAdvisor        api.SecurityAdvisor
	chanState         *channelState
	disSecAdap        *discoverySecurityAdapter
	mcs               api.MessageCryptoService
	stateInfoMsgStore msgstore.MessageStore
	certPuller        pull.Mediator
}
```

## 2.2 privateHandlers 字段
`privateHandlers map[string]privateHandler`
```go
type privateHandler struct {
	support     Support
	coordinator privdata2.Coordinator
	distributor privdata2.PvtDataDistributor
}
```

## 2.3 chains 字段
`chains          map[string]state.GossipStateProvider`
定义是 gossip/state/state.go 下的 GossipStateProvider interface，注释说 是~~通过状态复制与发送请求到其他节点来？？？~~获取缺失 block 的接口。实际实现是 GossipStateProviderImpl struct
```go
type GossipStateProvider interface {
	AddPayload(payload *proto.Payload) error

	// Stop terminates state transfer object
	Stop()
}
```

## 2.3.1 GossipProviderImpl struct 
```go
type GossipStateProviderImpl struct {
	// Chain id
	chainID string

	mediator *ServicesMediator

	// Channel to read gossip messages from
	gossipChan <-chan *proto.GossipMessage

	commChan <-chan proto.ReceivedMessage

	// Queue of payloads which wasn't acquired yet
	payloads PayloadsBuffer

	ledger ledgerResources

	stateResponseCh chan proto.ReceivedMessage

	stateRequestCh chan proto.ReceivedMessage

	stopCh chan struct{}

	done sync.WaitGroup

	once sync.Once

	stateTransferActive int32
}

```
## 2.4 leaderElection 字段
`leaderElection  map[string]election.LeaderElectionService`
定义是 gossip/election/election.go 下的 LeaderElectionService interface，该接口是用来实现 leader election algorithm 的，实际实现是 leaderElectionSvcImpl struct 

```go
type LeaderElectionService interface {
	// IsLeader returns whether this peer is a leader or not
	IsLeader() bool

	// Stop stops the LeaderElectionService
	Stop()

	// Yield relinquishes the leadership until a new leader is elected,
	// or a timeout expires
	Yield()
}
```

### 2.4.1 leaderElectionSvcImpl struct
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
## 2.5 deliveryService 字段
`deliveryService map[string]deliverclient.DeliverService`
定义是 core/deliverservice/deliverclient.go 下的 DeliverService interface，该接口用来与 OS 进行交流以获取新的 blocks 并把它们发送到 commiter service；实际实现是 deliverServiceImpl struct 
```go
type DeliverService interface {
	StartDeliverForChannel(chainID string, ledgerInfo blocksprovider.LedgerInfo, finalizer func()) error

	StopDeliverForChannel(chainID string) error

	// Stop terminates delivery service and closes the connection
	Stop()
}
```
### 2.5.1 deliverServiceImpl struct 
该结构体是对 delivery service 的实现，维护与 OS 的连接/ 对blocks provider 的映射
```go
type deliverServiceImpl struct {
	conf           *Config
	blockProviders map[string]blocksprovider.BlocksProvider
	lock           sync.RWMutex
	stopping       bool
}
```

## 2.6 deliveryFactory 字段
`deliveryFactory DeliveryServiceFactory`
定义是 gossip/service/gossip_service.go 下的 DeliveryServiceFactory insterface，工厂模式下对 delivery service 实例的创建与初始化；实际实现是 deliveryFactoryImpl struct
```go
type DeliveryServiceFactory interface {
	// Returns an instance of delivery client
	Service(g GossipService, endpoints []string, msc api.MessageCryptoService) (deliverclient.DeliverService, error)
}

type deliveryFactoryImpl struct {
}
```

## 2.7 lock 字段
`lock	sync.RWMutex`
golang 下的读写u锁

## 2.8 mcs 字段
`mcs             api.MessageCryptoService`
定义是 gossip/api/crypto.go 下的 MessageCryptoService interface，MessageCryptoService 是在 gossip 组件 与 peer 加密层之间的协议（contract）， MessageCryptoService 被 gossip 组件用来验证与授权远程 peer 与它们所发送的数据，也用来验证从 OS 收到的区块；实际实现是 peer/gossip/mcs.go 下的 mspMessageCryptoService struct
```go
type MessageCryptoService interface {
	GetPKIidOfCert(peerIdentity PeerIdentityType) common.PKIidType

	VerifyBlock(chainID common.ChainID, seqNum uint64, signedBlock []byte) error

	Sign(msg []byte) ([]byte, error)

	Verify(peerIdentity PeerIdentityType, signature, message []byte) error

	VerifyByChannel(chainID common.ChainID, peerIdentity PeerIdentityType, signature, message []byte) error

	ValidateIdentity(peerIdentity PeerIdentityType) error

	Expiration(peerIdentity PeerIdentityType) (time.Time, error)
}
```
### 2.8.1 mspMessageCryptoService struct
```go
type mspMessageCryptoService struct {
	channelPolicyManagerGetter policies.ChannelPolicyManagerGetter
	localSigner                crypto.LocalSigner
	deserializer               mgmt.DeserializersManager
}
```
## 2.9 peerIdentity 字段
`peerIdentity	[]byte`
peer 身份

## 2.10 secAdv 字段
`secAdv          api.SecurityAdvisor`
定义是 gossip/api/channel.go 下的 SecurityAdvisor interfac，SecurityAdvisor 定义了一个提供安全性和身份相关功能的外部辅助对象；实际实现是 peer/gossip/sa.go 下的 mspSecurityAdvisor struct
```go
type SecurityAdvisor interface {
	OrgByPeerIdentity(PeerIdentityType) OrgIdentityType
}
```
### 2.10.1 mspSecurityAdvisor struct
```go
type mspSecurityAdvisor struct {
	deserializer mgmt.DeserializersManager
}
```



