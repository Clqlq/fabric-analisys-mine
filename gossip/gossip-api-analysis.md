## gossip/api/
三个文件： 
	
	channel.go 
	crypto.go 
	subchannel.go
---
### channel.go
暂时不写
### subchannel.go
暂时不写
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
	// 如果身份无效，撤销，过期则返回错误
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

// PeerSuspector 返回一个有给定 indentity 的 peer 是否被怀疑
// 将被撤销，或者它的 CA 将被撤销
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


















