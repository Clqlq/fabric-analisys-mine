## gossip/api/
 
代码结构：
```go
/api
|	|- api_test.go
|	|- channel.go
|	|- crypto.go
|	|- subchannel.go

```

---
### channel.go


```go
// SecurityAdvisor 定义了一个提供安全性和身份相关功能的外部辅助对象
type SecurityAdvisor interface {
	// OrgByPeerIdentity 返回 给定 peer 对应的 OrgIdentityType
	// 出错则返回 nil　
	// 此方法不验证peerIdentity
	// 该验证应该在执行流程期间适当地进行
	OrgByPeerIdentity(PeerIdentityType) OrgIdentityType
}

// ChannelNotifier 由 gossip component 实现
// 用于 peer layer　通知 goosip component 一个　JoinChannel 事件
type ChannelNotifier interface {
	JoinChannel(joinMsg JoinChannelMessage, chainID common.ChainID)
}

// JoinChannelMessage是声明一个通道的成员列表的创建或变更的消息
// 并且是在对 peers 之间进行 gossip 的消息
type JoinChannelMessage interface {

	// SequenceNumber 返回 JoinChannelMessage 源自的配置块的序列号
	SequenceNumber() uint64

	//　Members　返回通道下的组织
	Members() []OrgIdentityType

	//　AnchorPeersOf　返回给定组织的所有 anchor peer
	AnchorPeersOf(org OrgIdentityType) []AnchorPeer
}

// AnchorPeer 是一个 anchor peer　的身份与端点
type AnchorPeer struct {
	Host string // Host is the hostname/ip address of the remote peer
	Port int    // Port is the port the remote peer is listening on
}

// OrgIdentityType 定义了一个组织的身份
type OrgIdentityType []byte

```
---

### subchannel.go

```go
// RoutingFilter 定义了哪些 peers　应该收到确定的消息
//　或者哪些 peers　是足够合格可以收取某一确定的消息
type RoutingFilter func(peerIdentity PeerIdentityType) bool

// CollectionCriteria describes a way of selecting peers from a sub-channel
// given their signatures and whether they are from our organization
//　CollectionCriteria　描述了一种通过给定对方的签名以及是否来自我们组织的信息
// 来从子通道中选择 peers　的方式
type SubChannelSelectionCriteria func(signature PeerSignature, isFromOurOrg bool) bool

// RoutingFilterFactory 定义了一个对象
// 通过给定一个 CollectionCriteria 和一个 channel
// 该对象便能断言哪些 peers　应该知道与 CollectionCriteria 相关的数据
type RoutingFilterFactory interface {
	// Peers returns a RoutingFilter for given chainID and CollectionCriteria
	Peers(common.ChainID, SubChannelSelectionCriteria) RoutingFilter
}

```

---
### crypto.go
```go

// MessageCryptoService 是在 gossip 组件 与 peer 的加密层之间的协议（contract）
// MessageCryptoService 被 gossip 组建用来验证与授权远程 peer 与它们所发送的数据
// 也用来验证从排序服务收到的区块
type MessageCryptoService interface {

	// GetPKIidOfCert 返回了 peer 身份的 PKI-ID
	// 如果发生错误，该方法则返回 nil
	// 该方法并不验证 peerIndentity，验证过程应该在 execution flow 中完成
	GetPKIidOfCert(peerIdentity PeerIdentityType) common.PKIidType

	// 如果区块已经被正确签名了的话，VerifyBlock 返回 nil
	// 声明的 seqNum 是包含在该 block 的 header 中的序列号
	// 否则则返回错误
	VerifyBlock(chainID common.ChainID, seqNum uint64, signedBlock []byte) error

	// Sign 用该 peer 的 signing key 对 msg 签名
	// 如果没错的话则输出该签名
	Sign(msg []byte) ([]byte, error)

	// Verify 使用 peer 的 verification key 来验证该签名
	// 是否是一个合法的签名
	// 如果成功的话则返回 nil
	// 如果 peerIndentity 是空的话则验证失败
	Verify(peerIdentity PeerIdentityType, signature, message []byte) error

	// VerifyByChannel 检查在一个 peer 的 verification key 下
	// 该信息的签名是否有效，但也在特定通道的上下文中
	// 如果验证成功，Verify 返回 nil
	// 如果 peerIndentity 是空则验证失败
	VerifyByChannel(chainID common.ChainID, peerIdentity PeerIdentityType, signature, message []byte) error

	// ValidateIdentity 验证远程 peer 的身份
	// 如果身份无效／撤销／过期则返回错误
	/// 否则返回 nil
	ValidateIdentity(peerIdentity PeerIdentityType) error

	// Expiration 返回：
	// - 该 indentity 到期的时间, nil
	//   如果它会到期的话
	// - 一个零值 time.Time, nil
	//   如果它不会到期的话
	// - 一个零值, error 
	//   如果它不能判断该身份是否会到期
	Expiration(peerIdentity PeerIdentityType) (time.Time, error)
}

// PeerIdentityType 是 peer 的 certificate
type PeerIdentityType []byte

// PeerSuspector 返回一个有给定 indentity 的 peer 是否
// 被怀疑将被撤销，或者它的 CA 将被撤销
type PeerSuspector func(identity PeerIdentityType) bool

// PeerSecureDialOpts 返回 gRPC DialOptions，用来
// 在和远程 peer 终端通信时用于连接级别安全性
type PeerSecureDialOpts func() []grpc.DialOption

// PeerSignature 定义了一个 peer 对特定信息的签名
type PeerSignature struct {
	Signature    []byte
	Message      []byte
	PeerIdentity PeerIdentityType
}

```


















