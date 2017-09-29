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

	// Send 发送了一个 message　到远程 peer
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





















