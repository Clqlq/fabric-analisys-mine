## Fabric 一致性实现

看了一下文档中的 /Hyperledger Fabric Model/Consensus 部分，感觉什么都没讲，就是最后一句话有点意思：
> To conclude, consensus is not merely limited to the agreed upon order of a batch of transactions, but rather, it is an overarching characterization that is achieved as a byproduct of the ongoing verifications that take place during a transaction’s journey from proposal to commitment.

但核心还是 transaction 的排序服务，也就是代码中的`/fabric/orderer`