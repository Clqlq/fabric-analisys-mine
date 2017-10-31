## 概述

- 一个 data item 永远不会到达不是属于与该 data item 关联的通道的 node；

- 提出的分发网络将会将信息分发到所有的 peers，除此之外还会进行一些后台运行，以保证所有的 peers 收到相同的 data／到达相同的 state；

---

- ordering service 与　peers　的分隔，有助于维护 ledger 中进行复杂数据的分发与 Byzantine 环境下的验证发布

- Fabric gossip 的主要目标是：
	- 使得所有的 peers 有 ledger 的相同复制，缓解所有 peers 直接连向ordering service 请求 ledger blocks 的问题
	- 在不涉及 ordering service 的情况下，将 ledger 发送至新连接进的 peers

- Gossip 组件可以在不同 ordering service 的实现下使用

--- 
- order updates to peers

	- 特定的 peers 将会通过标准 Deliver() 协议与 ordering service 连接，这些 peers 将会把接收到的 batches 分发给其他 peers

	- 每一个 Org 会选择一个 peer 来代表它与 OS 进行连接，并把 batches 分发给同属同一 Org 的其他 peers

---
- state transfer / synchronization

	- 第一，对于错过少量 batch updates 的 peers，基于 gossip 的 state synchronization 机制将会保证，落后的 peer 将会从其他 peers 得到错过的 blocks

	- 第二，为了支持将 peers 加入已经开启运行一段时间的网络，应该提供一个基于 anti-entropy 的 state synchronization，用此机制来进行 peers 之间大量 data items 的传输

---
- Multichannel support
	- 新 channel 的建立是通过 client SDK 发送一个 CONFIGURATION transaction　背书之后发送到 OS
	- channel 创建时，指明了构成 channel 的 Org 列表
	- 一旦 channel 创建成功，client SDK 能够指导属于相关 Org 的特定 peers 加入新建立的 channel
	- 正加入的 peers 会使用 gossip 组件，向属于该 channel 的所有 Org 进行广播，表明其已加入 channel
	- 所有加入 channel 的 peers 都需要知道对方的存在
	- 代表 Org 与 OS 进行连接的 peer 将会收到关于 哪些 peers 属于哪个 channel 的信息，并将代表它们调用 Deliver() 协议
	- OS 对应地应该保证该 peer 确实属于某个参与进该 channel 的 Org，并将 bathces 发送到与之连接的 peer
	- 与之连接的 peer 对应地，将会将收到的 bolcks 分发到加入该 channel 的 peers
	- 
	- 一旦 channel membership 建立，gossip componet 将会根据常规算法进行操作

- ~
	- 为了实现 peer 严格保证所发消息都控制在对应 channel 中，每一个 peer 加入 channel 时，都必须最新的 channel configuration，以判断哪些组织在 channel 中(对于新建的 channel，其配置及 Genesis block)
	- 每个 channel 的 gossip 都将以与标准 gossip 相同的方式工作
	- 注意 state synchronization 可以在不同 Org 的 peers 中进行，只要它们同属同一个通道
	- peer 加入与被移除通道都将通过 configuration transaction 来完成

---

- Authentication and membership management
	- peer 的身份称为 PKI-ID
	- 有两种 message 的发送方式
		- Gossip（从一个 peer 发往一组 peers，意味着 message 可以在 peers 中被中继）
			- 必须包含 peer 的 PKI-ID 
			- 必须被该 peer 签名
			- 能被该 peer 的证书验签
		- point-to-point
			- 点对点间发送的不是gossip 的 message,并没有被签名，因为有这样的假设：在生产环境中，peers 的 TLS 层是激活的,以此来保证安全性
	- 在 peers 间被点对点传输，但却不被 peer 签名也不是 gossip 的 message 只会是 包含 Ledger block 的message，该 message 被 OS 签名
	- peer 的成员视图（membership view）(即 peer 所知的 peers)由如下方式构建：
		- 每一个 peer 都周期性地 gossip 一个特定的 message —— AliveMessage,该 message 包含：
			- PKI-ID
			- Endpoint（host + ":" + port）
			- Metadata
			- PeerTime（包含）：
				- Peer’s incarnation time (startup time)
				- 单调递增计数器，在 AliveMessage 的每次分发时递增
			- 对上述字段的 signature
			- peer 的 certificate（optional）
	- peer 的证书只会在当 peer 开始启动的一段有限的时间内被包含在 message 中,过了时间则不再被包含
	- 当一个 peerA 从其他 peers 收到 message，peerA 应该用在这之前就收到过的证书对 peer 的签名验签（这就是为什么在 peer 启动的一段时间内 AliveMessage 会包含 peer certificate 的原因，目的就是为了使得其他 peers 能够验证其签名）
	- 没有收到其他 peers certificate 的 Peers 可以通过一种周期性的分发机制来获取其他 peer的证(##（具体见后面）##
	- 当 peers 收到一个来自 remote peer 的 AliveMessage，它都将通过 AliveMessage 中的 endpoint 字段与之建立连接
	- ~
	- ~
	- ~
	- 在生产环境中，已经假设 peers 使用了 mutual TLS，TLS 连接的每一侧都有一个合法的 TLS certificate
	- 当一个 peerA 和 其他 peerB 第一次交流时，peerA 与 peerB 执行一个 handshake，以验证 peerB 有该 TLS certificate 对应 证书，因此将该 TLS session（回话）与该 fabric membership identity 绑定在一起
	- handshake 是对称与简单的
	- ~
	- ~
	- ~
	- 每个 peer 发往 remote peers 的第一个 message 是 ConnEstablish message，该消息包括：
		- 对该 peer 的 TLS certificate 的 hash 的签名
		- 该 peer 的 PKI-ID
		- fabric MSP certificate
	- 每个 peer 反过来：
		- 接收该 message
		- 提取该 remote peer 的 certificate 并对其 hash
		- 验证 ConnEstablish 中对该 hash 的签名
		- （如果验证失败，则 peer 拒绝该连接）
	- Handling channel membership changes：当 gossip component 初始化时，它给了一个 channel 验证策略的实现，该策略的实现可以决定一个 remote peer 与 其所属的 Org 之间的 mapping
		- 上述实现：维护每个 peer PKI-ID 与其 root Org CA 的 mapping（本地 Ledger 的 last configuration 含有 channel 所含 Org（由 root Org CA 代表）的信息
	- OS 发送的 batches 包含了该通道 last configuration block 的序列号。当一个 peerA 从 peerB 收到一个 block 时，peerA 查看该 block 的序列号与其自己所有的 Ledger 的最新的序列号，然后查询last configuration block 的序列号
		- 如果写在接收到的 block 中的 last configuration block 的序列号，比本地 Ledger 最新 block 的序列号要搞，则 peerA 将收到的 data block 放在内存中，并不会转发
		- 反之，则意味着该账本的最新 configuration 已经更新，接收到的 block 可以安全地发往合法的（处在同一通道中的）其他 peers

---

- Resulting Capabilities Needed
	- 从一个 source 到多个 destination 的有效的数据分发
	- peer-to-peer gossip
	- Pub / sub like features to support multi-channels
	- Membership – keep an updated list of nodes that are alive and connected with associated metadata

- Initial API
	- Gossip() —— 发送一个 message 到特定 gossip group 中的所有 nodes
	- Accept() —— 从其他 peer 接收一个 message
	- GetPeers() —— 获取当前的成员视图
	- UpdateMetadata() —— 更新该 node 的 metadata
	- Stop() —— 停止该 gossip com破net


## 实现
- 提供的功能：
	- Discovery and Membership —— 持续跟踪 membership 的状态（alive？dead？）
	- Broadcast —— 将一个 message 从单一源发往所有成员
	- Background activity —— 同步所有 nodes 的状态
- Gossip-based broadcast complementary modes of operation:
	- Push：一开始粗暴地分发 message
	- Push-pull：与邻居交换信息以实现同步
	- Anti-entropy：偶尔与邻居交换信息以同步 global state
- 基于 gossip 的广播实现：一个 peer 接收一个 message，随机选择一些 peers 然后将 message 发给它们
- 在 peers 间同步状态：一个 peer 周期性地与其他 peers 同步比较信息，然后进行状态同步（pull）
- 每一个 peer 都维护了 membership，以使得 peer 能从中随机选择一些 peers 并将 message 发送过去（不用维护复杂的连接，所以这种方法具有很好的鲁棒性）

- Membership —— discovery
	- 每一个 node 在开启时都收到一个 configuration parameter，包含了 a set of peers it should try to connnect
	- 


## Leader election




