## 代码结构
```
/algo
|	|- pull.go
```
---

## pull.go

```
PullEngine 是执行 pull-based gossip 的对象，并且维护那些由字符串代表身份的条目（items）的内部状态。
该协议如下：
1) Initiator 发送一个带有特定 NONCE 编号的 Helloo message 到一个远程 peers　的集合
2) 每一个远程 peer　都回复一个带有它所拥有 items　的摘要的 message，并且也带上了 NONCE
3) Initiator 检查收到的 NONCES 的有效性，收集收到的 message　的摘要。然后将想要从远程 peer　得到的 item　的 id　摘要与 NONCE 打包成一个 request，将 request 发生到对应 peer
4) 如果 peer 仍然持有 NONCE 与相关的 items，则返回被请求的 items 与 NONCE

    Other peer				   			   Initiator
	 O	<-------- Hello <NONCE> -------------------------		O
	/|\	--------- Digest <[3,5,8, 10...], NONCE> -------->     /|\
	 |	<-------- Request <[3,8], NONCE> -----------------      |
	/ \	--------- Response <[item3, item8], NONCE>------->     / \
```

- `type DigestFilter func(context interface{}) func(digestItem string) bool` DigestFilter 基于远程 peer 发送来的 hello 或 request 的 message 的内容，过滤将要发送到远程 peer 的 digests

- PullAdapter
```go

// PullEngine 需要 PullAdapter 来发送 messages 到远程 PullEngine 实例
// PullEngine 应该在远程 PullEngine　发送来的 OnHello / OnDigest / OnReq / OnRes 对应 message 到达时被调用
type PullAdapter interface {
	// SelectPeers 返回一个 peers　切片，包含了哪些 peers　在初始化该协议时会被涉及
	SelectPeers() []string

	// Hello 发送一个 hello message 来初始化该协议
	// 并且返回一个 NONCE，该 NONCE 应该在 digets　message 
	// 中被返回
	Hello(dest string, nonce uint64)

	// SendDigest sends a digest to a remote PullEngine.
	// The context parameter specifies the remote engine to send to.
	// SendDigest 发送一个 digest 到远程 PullEngine
	// 
	SendDigest(digest []string, nonce uint64, context interface{})

	// SendReq sends an array of items to a certain remote PullEngine identified
	// by a string
	SendReq(dest string, items []string, nonce uint64)

	// SendRes sends an array of items to a remote PullEngine identified by a context.
	SendRes(items []string, context interface{}, nonce uint64)
}



```



