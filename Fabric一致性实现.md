## Fabric 一致性实现

看了一下文档中的 /Hyperledger Fabric Model/Consensus 部分，感觉什么都没讲，就是最后一句话有点意思：
> To conclude, consensus is not merely limited to the agreed upon order of a batch of transactions, but rather, it is an overarching characterization that is achieved as a byproduct of the ongoing verifications that take place during a transaction’s journey from proposal to commitment.

但核心还是 transaction 的排序服务，也就是代码中的`/fabric/orderer`

### 1. CAPM CAP Theorem
分布式系统下的 CAPM 理论
consistency,  avaliability, partition tolerance 理论

以后要做到 共识算法 可插拔

Temporal Order：


区块链的分布式：每一个节点都做同一件事情，不同于以往的分布式系统（不同的节点做不同的事情）。

CFT( Crash faults ): Paxos, RAFT, Zookeepr AB
Non-crash faults:  例如BFT（拜占庭问题）

使用 CFT性能会高很多，但使用了 NBFT的假设，假设太强

不是先记账，而是先取得共识



|CFT|XFT|BFT|
的表格



上面啥都没说，没有价值
---




