## Fabric 中基于Kafka的排序服务
### 1. 介绍
我们使用 Kafka 来支持在CFT（crash fault tolerant）场景下的排序服务以及对多链的支持。排序服务由这些东西组成：

- (1). 一个Kafka 的集群以及它自己对应的ZooKeeper ensemble（这是啥？）
- (2). 排序服务节点的集合，这些节点在**排序服务**的客户端 与 Kafka 集群之间



![Fig. 1. An ordering service, consisting of 5 ordering service nodes (OSN-n), and a Kafka cluster. The ordering service client can be connected to multiple OSNs. Note that the OSNs do not communicate with each other directly.](/fabric_kafka_images/fig1.png)



这些排序节点（ordering service nodes, OSNs）：

- (1). 进行客户端认证
- (2). 允许客户端写一个**链**，或者使用一个简单的接口读一个**链**
- (3). 它们也进行 transaction 过滤，也可以对 configuration transactinos 做验证，不论是对已有**链**的 **reconfig** 还是新建一个**链**

### 2. Where does Kafka come in?(不知道怎么翻译)
在 Kafka 中，Messages (records) 被写入**主题分区** (topic partition)。一个 Kafka 集群可以有多个**主题** (topics)，每个**主题**可以有多个**分区** (partitions)，如图2。每个分区是一个有序的，不可变的记录序列，并且被不断附加。

![Fig. 2. A topic that consists of three partitions. Each record in a partition is tagged with an offset number. (Diagram taken from the Kafka website.)](/fabric_kafka_images/fig2.png)

Solution 0. 假设每一条**链**都有一个separate （独立的？分隔的？）**分区**。每当OSNs 完成了 客户端的验证与对 transaction 的过滤，它们都能把输入进来的 属于某个特定**链**的 客户端的 transactions 中继 (relay) 到该链对应的**分区**上。然后，他们可以使用该**分区**，并返回所有OSNs中常见的 transactions 的有序列表。

![Fig. 3. Assume all client transactions here belong to the same chain. OSNs relay incoming client transactions to the same partition in the Kafka cluster. They can then read from that partition and get all the transactions that were posted to the network in an order that will be the same for all OSNs.](/fabric_kafka_images/fig3.png)

结束了吗？接着看

在上面的例子中，每一个 client transaction 都是一个独立的 block；OSN 把每一个 transaction 打包进一个 Block 类型的 message 中，并对其分配一个值，该值与 Kafka 集群分配给它的**偏移值** (offser number) 相同，并对该 block 签名，以便审计 (for auditability)。任何客户端发来的 `Deliver RPCs`消息，都由在该**分区**上设定的一个 Kafka consumer 来回复，Kafka consumer 会寻找特定的 block/offset number。

**Problem 1.** 但是我们考虑一下 transaction 被传入的速率，比如说 1K transactions 每秒。那么 排序服务 (ordering service) 则必须每秒生成 1K个签名，并且 客户端在接受侧 (receiving side) 也必须每秒验证 1K个签名。众所周知，签名与验签名是很耗费计算能力的 (expensive)，所以这样做不可行

**Solution 1.** 所以于其把每一个 transaction 放在单独的 block 中，我们可以批量的把 transactions 放入 block 中。所以以上面的情况为例，每 1K个transactions 只需要 排序服务 做一次签名，客户端 做一次验证。

**Problem 2.** 但是如果 transactions 进入的速率不是恒定的呢？假设 排序服务 刚发出了一批 1K的transaction，现在在内存中有999个transactions，还在等待下一个 transaction 到来以形成新的 一批 (new batch)，但是一个小时过去了，都没有新的 transaction 被推送到 排序服务。那么先前到达的999个 transactions 的交付时间会有一个不可接受的延时

**Solution 2.** 我们意识到，我们需要一个批处理计时器 (a batch timer)。当新的一批 transactions 的第一个 transation 进入时，设定计时器。接着，无论是我们达到了一个block 所能具有的消息的最大数量 (batchSize)，还是当计时器计时器当期 (batchTimeout)，我们都可以切分出一个新block (cut a block)。

**Problem 3.** 然而，基于时间的 block 的切分需要调用在 service 节点中的协调 (coordination)。基于消息数量的 block 的切分是容易的，比如说，每当你在一个分区中读取了2条（你设定的 batchSize）消息，你就可以切出一个块。这个分区对于所有的OSN来说都是一样的，所以无论你接触哪个OSN，都可以保证客户端能够获得相同的块序列。考虑下面的情况，当你设定了一个 batchTimeOut = 1s， 排序服务中有两个 OSN。一个 block 刚被切分出来，而且接着又通过 OSN1 进来了 (get posted)一个新的 transaction，该transaction被推送到到分区中。OSN2在 t=5s 时读到该交易，然后设置了一个将在 t=6s 时到期的计时器。OSN1 在 t=5.6s 时从该**分区**读到了该 transaction，设置了对应的计时器。接着，第二个 transaction 被推送到 (be posted)到该**分区**（其实我对怎么推送的？通过不同的OSN来推送是否有差别？等问题并不明白），OSN2 在 t=6.2s 时读到了它，而 OSN1 在 t=6.5s 时读到了它。现在出现了这样一种情况，OSN1 新切出的 block 包含了这两个 transaction，而OSN2切出的 block 只包含第一个 transaction，于是它们生成了两个不同的 block，区块链上出现了分歧，这是不能接受的。

**Solution 3.** 因此，基于时间的 block 的切分，需要一个明确的协调信号。让我们假设每一个 OSN 在切出一个新的 block 时都会给**分区**推送一个“到切第 X 块的时间啦！”的消息（其中 X 是一个整数，对应在序列中的下一个 block），并且 OSN 在从一个**分区**读到这个消息之前，它是不会切出新的 block 的。注意，读取到的消息（一下简称为 TTC-X）并不一定是它自己发出的，如果每个 OSN 都在等待自己的 TTC-X，那么会再次出现分歧。于是，每一个 OSN 在收集到了 batchSize 条消息或者接收到第一个 TTC-X 消息后，都会切出一个 新的 block。这意味着，后续所有的具有相同 X 值的 TTC-X都会被忽略。如图4所示。

![FFig. 4. OSNs post incoming transactions and TTC-X messages to the partition.](/fabric_kafka_images/fig4.png)

我们现在已经找到了切出新 block 的方法，不论是基于 transaction 数量还是基于时间我们都能保证在所有 OSN 中，block 序列的一致性。

**Problem 4.** 和把每个 transaction 放入独立的 block 情况不同的是，现在 block 的编号不会转换为 Kafka 的该分区的偏移号。因此，如果**排序服务**从块接收到一个 Deliver 请求，比如说5，那么它根本就不知道 consumer 应该去找哪个偏移值（对这段我理解不是很懂，是指如果要找到对应的 tx，而不能根据 offset 找到吗？）

**Solution 4a.** 也许我们可以使用 Block 消息类型中 Metadata 域，并且可以让 OSN 记下该 block所携带的（交易的？）偏移值范围（比如，Block 4的 Metadata 包含内容：“offset: 63-64”）。那么如果访问 block 9的客户端想要从 block5开始获取流，则它将通过将起始号设置为65来发出一个 Deliver RPC，并且 OSN 可以从偏移值65开始 replay 分区日志，当 tx 数量先达到 batchSize或者先找到一个 TTC-X时，切下一个 block（这个切下到底是个什么操作？？？）。这个方法有两个问题。第一，我们如果以 block 编号为参数的话，就违反了 Deliver API 的协议；第二，如果一个客户端错过了一堆 block，只是想获得随机的 block X（在不拥有 block X-1 的情况下），或者最近生成的 block（通过在 SeekInfo 消息中的 "NEWEST" 参数）？客户端不知道该通过 Deliver 调用该传递的正确的 offset 值是多少，OSN也不知道。

**Solution 4b.** 因此每一个 OSN 都需要对每条**链**维护一个表，用来映射每个 block 到它们所包含的第一个 offset，如表1所示。注意，这就意味着，如果一个 OSN 的内建查询表不包含所请求的 block 号的话，它就不能回应（accommodate）一个 Deliver 请求。


|Block Number|Offset Number|
|:----------:|:-----------:|
|	...		 |		...	   |
|4			 |63		   |
|5			 |65		   |
|	...		 |	...		   |


这个查询表就使得我们不再需要 block metadata，并且可以让客户端在 Deliver 请求中指明正确的 offset。OSN 将请求的 block 编号转换为正确的 offset 值，并设置了一个 consumer 来查找它。现在，我们通过维护一些表，克服了上面所说的问题。但又出现了两个问题。
**Problem 5.** 第一，注意，每当 OSN 接收到一个 Deliver 请求，它必须从请求的 block 号开始从该**分区**检索所有的消息，将它们打包成 block 并签名。这一打包与签名过程在每一个 replay 请求中都会重复，并且耗费昂贵。**Problem 6.** 第二，因为我们在分区中有我们需要跳过的冗余的消息，因此回复一个 Deliver 请求并不像设置一个 consumer，寻找请求的 offset，replay所有的记录那样简单；查询表会在任意时刻被查询，交付逻辑开始变得越来越复杂，并且对查询表的检查会增加延迟。我们先不考虑第二个问题，针对第一个问题，为了解决它，我们需要在创建 blocks 时维持它们（peersist these blocks as we create them），当我们 replaying（不知道这个怎么翻译？？？）时，我们只需要传输已维持着的 blocks（transmit the persisted blocks）。

**Solution 5a.** 为了解决这个问题，假设我们要全部进入Kafka（？？？），我们创建另一个**分区** - 让我们称之为分区1，这意味着我们目前为止保持引用的分区是分区0。现在，每当 OSNs 切出一个 block，它们都会将其推送（post)到分区1，并且所有的 Deliver 请求都将由这个分区提供服务（served by that partitioin？？怎么翻译妥当？？）。
















