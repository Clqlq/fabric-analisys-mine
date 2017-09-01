## Fabric 中基于Kafka的排序服务
### 1. 介绍
我们使用 Kafka 来支持在CFT（crash fault tolerant）场景下的排序服务以及对多链的支持。排序服务由这些东西组成：

- (1). 一个Kafka 的集群以及它自己对应的ZooKeeper ensemble（这是啥？）
(2). 排序服务节点的集合，这些节点在 排序服务的客户端 与 Kafka 集群之间

Fig1

这些排序节点（ordering service nodes, OSNs）：

- (1). 进行客户端认证
(2). 允许客户端写一个 链，或者使用一个简单的接口读一个 链
(3). 它们也进行 transaction 过滤，也可以对 configuration transactinos 做验证，不论是对已有 链 的 reconfig 还是新建一个 链

### 2. Where does Kafka come in?(不知道怎么翻译)
在 Kafka 中，Messages (records) 被写入主题分区 (topic partition)。一个 Kafka 集群可以有多个主题 (topics)，每个主题可以有多个分区 (partitions)，如图2。每个分区是一个有序的，不可变的记录序列，并且被不断附加。

Fig2

Solution 0. 假设每一条 链 都有一个separate （独立的？分隔的？）分区。每当OSNs 完成了 客户端的验证与对 transaction 的过滤，它们都能把输入进来的 属于某个特定链的 客户端的 transactions 中继 (relay) 到该链对应的 分区上。然后，他们可以使用该分区，并返回所有OSNs中常见的 transactions 的有序列表。

Fig3

结束了吗？接着看

在上面的例子中，每一个 client transaction 都是一个独立的 block；OSN 把每一个 transaction 打包进一个 Block 类型的 message 中，并对其分配一个值，该值与 Kafka 集群分配给它的偏移值 (offser number) 相同，并对该 block 签名，以便审计 (for auditability)。任何客户端发来的 `Deliver RPCs`消息，都由在该 分区上设定的一个 Kafka consumer 来回复，Kafka consumer 会寻找特定的 block/offset number。

**Problem 1.** 但是我们考虑一下 transaction 被传入的速率，比如说 1K transactions 每秒。那么 排序服务 (ordering service) 则必须每秒生成 1K个签名，并且 客户端在接受侧 (receiving side) 也必须每秒验证 1K个签名。众所周知，签名与验签名是很耗费计算能力的 (expensive)，所以这样做不可行

**Solution 1.** 所以于其把每一个 transaction 放在单独的 block 中，我们可以批量的把 transactions 放入 block 中。所以以上面的情况为例，每 1K个transactions 只需要 排序服务 做一次签名，客户端 做一次验证。

**Problem 2.** 但是如果 transactions 进入的速率不是恒定的呢？假设 排序服务 刚发出了一批 1K的transaction，现在在内存中有999个transactions，还在等待下一个 transaction 到来以形成新的 一批 (new batch)，但是一个小时过去了，都没有新的 transaction 被推送到 排序服务。那么先前到达的999个 transactions 的交付时间会有一个不可接受的延时

**Solution 2.** 我们意识到，我们需要一个批处理计时器 (a batch timer)。当新的一批 transactions 的第一个 transation 进入时，设定计时器。接着，无论是我们达到了一个block 所能具有的消息的最大数量 (batchSize)，还是当计时器计时器当期 (batchTimeout)，我们都可以切分出一个新block (cut a block)。

**Problem 3.** 


















