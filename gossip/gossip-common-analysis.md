## 代码结构
```
/common
|	|- common.go
|	|- metastate.go
```
---
## common.go
```go

// PKIidType 定义了持有 PKI-id 的类型
// PKI-id 是一个 peer 的安全身份
type PKIidType []byte

// MessageAcceptor是一个谓词
// 用于确定创建MessageAcceptor实例的订阅者感兴趣的消息。
type MessageAcceptor func(interface{}) bool

// Payload 定义了一个包含账本区块的对象
type Payload struct {
	ChainID ChainID // 区块所属通道的通道 ID
	Data    []byte  // 该消息的内容，可能已加密或签名
	Hash    string  // 消息的 hash
	SeqNum  uint64  // 消息的序列号
}

// ChainID 定义了一条链的身份表示
type ChainID []byte

// MessageReplacingPolicy 返回：
// MESSAGE_INVALIDATES 如果 this message 使 that 无效
// MESSAGE_INVALIDATED 如果 that 使 this message 无效
// MESSAGE_NO_ACTION 其他情况
type MessageReplacingPolicy func(this interface{}, that interface{}) InvalidationResult

// InvalidationResult 确定一条 message 将如何影响另一条 message
// 当该 message 被放入 gossip message 存储中时
type InvalidationResult int

const (
	// MessageNoAction 意味着 messages 之间没有关系
	MessageNoAction InvalidationResult = iota
	// MessageInvalidates 意味着 message 使其他 message 无效
	MessageInvalidates
	// MessageInvalidated 意味着其他 message 使该 message 无效
	MessageInvalidated
)

```
---
## metastate.go
```go
// NodeMetastate 存储了当前账本的高度信息
// 即最后一个i接收的块的序列号
type NodeMetastate struct {

	// 账本高度
	LedgerHeight uint64
}
```
- 两个函数
	- `func NewNodeMetastate(height uint64) *NodeMetastate {}`通过给定的账本高度创建新的 meta data
	- `func FromBytes(buf []byte) (*NodeMetastate, error) {}` 将字节数组编码成 meta data 结构体

- 三个方法
	- `func (n *NodeMetastate) Bytes() ([]byte, error) {}`　将 meta data 解码成字节数组
	- `func (n *NodeMetastate) Height() uint64 {}` 从 state 返回账本高度
	- `func (n *NodeMetastate) Update(height uint64) {}` 用新的账本高度更新 state














