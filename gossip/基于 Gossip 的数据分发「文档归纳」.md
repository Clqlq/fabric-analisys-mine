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
	- 







