## Fabric 中基于Kafka的排序服务
### 1. 介绍
我们使用 Kafka 来支持在CFT（crash fault tolerant）场景下的排序服务以及对多链的支持。排序服务由这些部分组成：

- (1). 一个Kafka 的集群以及它自己对应的ZooKeeper ensemble
- (2). 排序服务节点的集合，这些节点在排序服务的客户端 与 Kafka 集群之间


![Fig. 1. An ordering service, consisting of 5 ordering service nodes (OSN-n), and a Kafka cluster. The ordering service client can be connected to multiple OSNs. Note that the OSNs do not communicate with each other directly.](http://upload-images.jianshu.io/upload_images/7147675-f6c9a9ccb145b406.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





这些排序节点（ordering service nodes, OSNs）：
- (1). 进行客户端认证
- (2). 允许客户端写一个**链**，或者使用一个简单的接口读一个**链**
- (3). 它们也进行 transaction 过滤，也可以对 configuration transactinos 做验证，不论是对已有**链**的 **reconfig** 还是新建一个**链**

### 2. Where does Kafka come in?(译者：不知道怎么翻译)
在 Kafka 中，Messages (records) 被写入**主题分区** (topic partition)。一个 Kafka 集群可以有多个**主题** (topics)，每个**主题**可以有多个**分区** (partitions)，如图2。每个分区是一个有序的，不可变的记录序列，并且被不断附加。





![Fig. 2. A topic that consists of three partitions. Each record in a partition is tagged with an offset number. (Diagram taken from the Kafka website.)](http://upload-images.jianshu.io/upload_images/7147675-3562c94f599962b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Solution 0. 假设每一条**链**都有一个独立的**分区**。每当OSNs 完成了客户端的验证与对 transaction 的过滤，它们都能把输入进来的属于某个特定**链**的客户端的 transactions 中继 (relay) 到该链对应的**分区**上。然后，他们可以使用该**分区**，并返回所有OSNs中常见的 transactions 的有序列表。

![Fig. 3. Assume all client transactions here belong to the same chain. OSNs relay incoming client transactions to the same partition in the Kafka cluster. They can then read from that partition and get all the transactions that were posted to the network in an order that will be the same for all OSNs.](http://upload-images.jianshu.io/upload_images/7147675-bc9f388c1beeb16c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结束了吗？接着看

在上面的例子中，每一个 client transaction 都是一个独立的 block；OSN 把每一个 transaction 打包进一个 Block 类型的 message 中，并对其分配一个值，该值与 Kafka 集群分配给它的**偏移值** (offser number) 相同，并对该 block 签名，以便审计 (for auditability)。任何客户端发来的 `Deliver RPCs`消息，都由在该**分区**上设定的一个 Kafka consumer 来回复，Kafka consumer 会寻找特定的 block/offset number。

**Problem 1.** 但是我们考虑一下 transaction 被传入的速率，比如说 1K transactions 每秒。那么 排序服务 (ordering service) 则必须每秒生成 1K个签名，并且客户端在接受侧 (receiving side) 也必须每秒验证 1K个签名。众所周知，签名与验签是很耗费算力的 (expensive)，所以这样做不可行

**Solution 1.** 所以与其把每一个 transaction 放在单独的 block 中，我们可以批量的把 transactions 放入 block 中。所以以上面的情况为例，每 1K个transactions 只需要 排序服务 做一次签名，客户端 做一次验证。

**Problem 2.** 但是如果 transactions 进入的速率不是恒定的呢？假设排序服务刚发出了一批 1K 的transaction，现在在内存中有999个transactions，还在等待下一个 transaction 到来以形成新的一批 (new batch)，但是一个小时过去了，都没有新的 transaction 被推送到 排序服务。那么先前到达的999个 transactions 的交付时间会有一个不可接受的延时

**Solution 2.** 我们意识到，我们需要一个批处理计时器 (a batch timer)。当新的一批 transactions 的第一个 transation 进入时，设定计时器。接着，无论是我们达到了一个block 所能具有的消息的最大数量 (batchSize)，还是当计时器计时器当期 (batchTimeout)，我们都可以切分出一个新block (cut a block)。

**Problem 3.** 然而，基于时间的 block 的切分需要调用在 service 节点中的协调 (coordination)。基于消息数量的 block 的切分是容易的，比如说，每当你在一个分区中读取了2条（你设定的 batchSize）消息，你就可以切出一个块。这个分区对于所有的OSN来说都是一样的，所以无论你接触哪个OSN，都可以保证客户端能够获得相同的块序列。考虑下面的情况，当你设定了一个 batchTimeOut = 1s， 排序服务中有两个 OSN。一个 block 刚被切分出来，而且接着又通过 OSN1 进来了 (get posted)一个新的 transaction，该transaction被推送到到分区中。OSN2在 t=5s 时读到该交易，然后设置了一个将在 t=6s 时到期的计时器。OSN1 在 t=5.6s 时从该**分区**读到了该 transaction，设置了对应的计时器。接着，第二个 transaction 被推送到 (be posted)到该**分区**（译者：其实我对怎么推送的？通过不同的OSN来推送是否有差别？等问题并不明白），OSN2 在 t=6.2s 时读到了它，而 OSN1 在 t=6.5s 时读到了它。现在出现了这样一种情况，OSN1 新切出的 block 包含了这两个 transaction，而OSN2切出的 block 只包含第一个 transaction，于是它们生成了两个不同的 block，区块链上出现了分歧，这是不能接受的。

**Solution 3.** 因此，基于时间的 block 的切分，需要一个明确的协调信号。让我们假设每一个 OSN 在切出一个新的 block 时都会给**分区**推送一个“到切第 X 块的时间啦！”的消息（其中 X 是一个整数，对应在序列中的下一个 block），并且 OSN 在从一个**分区**读到这个消息之前，它是不会切出新的 block 的。注意，读取到的消息（以下简称为 TTC-X）并不一定是它自己发出的，如果每个 OSN 都在等待自己的 TTC-X，那么会再次出现分歧。于是，每一个 OSN 在收集到了 batchSize 条消息或者接收到第一个 TTC-X 消息后，都会切出一个 新的 block。这意味着，后续所有的具有相同 X 值的 TTC-X都会被忽略。如图4所示。


![Fig. 4. OSNs post incoming transactions and TTC-X messages to the partition.](http://upload-images.jianshu.io/upload_images/7147675-d900ec32d88c3465.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们现在已经找到了切出新 block 的方法，不论是基于 transaction 数量还是基于时间我们都能保证在所有 OSN 中，block 序列的一致性。

**Problem 4.** 和把每个 transaction 放入独立的 block 情况不同的是，现在 block 的编号不会转换为 Kafka 的该分区的偏移号。因此，如果**排序服务**接收到一个 Deliver 请求，比如说从 block 5 开始返回块，那么它根本就不知道 consumer 应该去找哪个偏移值（译者：对这段我理解不是很懂，是指如果要找到对应的 block，不能根据 相应的transaction 的 offset 找到吗？）

**Solution 4a.** 也许我们可以使用 Block 消息类型中 Metadata 域，并且可以让 OSN 记下该 block所携带的（交易的）偏移值范围（比如，Block 4的 Metadata 包含内容：“offset: 63-64”）。那么如果访问 block 4 （译者：原文是 block 9，但我觉得应该是写错了） 的客户端想要从 block 5 开始获取流，则它将通过将起始号设置为65来发出一个 Deliver RPC，并且 OSN 可以从偏移值65开始 replay 分区日志，当 transaction 数量先达到 batchSize或者先找到一个 TTC-X时，切下一个 block。这个方法有两个问题。第一，我们如果以偏移值为参数的话，就违反了 Deliver API 的协议，Deliver 要求以 block 编号为参数；第二，如果一个客户端错过了一堆 block，只是想获得随机的 block X（在不拥有 block X-1 的情况下），或者最近生成的 block（通过在 SeekInfo 消息中的 "NEWEST" 参数）客户端不知道该通过 Deliver 调用该传递的正确的 offset 值是多少，OSN也不知道。

**Solution 4b.** 因此每一个 OSN 都需要对每条**链**维护一个表，用来映射每个 block 到它们所包含的第一个 offset，如表1所示。注意，这就意味着，如果一个 OSN 的内建查询表不包含所请求的 block 号的话，它就不能回应（accommodate）一个 Deliver 请求。


|Block Number|Offset Number|
|:----------:|:-----------:|
|	...		 |		...	   |
|4			 |63		   |
|5			 |65		   |
|	...		 |	...		   |
`Table 1. Example lookup table corresponding to Fig. 4. A block number is mapped to the offset number of the first transaction that it should contain`

这个查询表就使得我们不再需要 block metadata，并且可以让客户端在 Deliver 请求中指明正确的 offset。OSN 将请求的 block 编号转换为正确的 offset 值，并设置了一个 consumer 来查找它。现在，我们通过维护一些表，克服了上面所说的问题。但又出现了两个问题。

**Problem 5.** 第一，注意，每当 OSN 接收到一个 Deliver 请求，它必须从请求的 block 号开始从该**分区**检索所有的消息，将它们打包成 block 并签名。这一打包与签名过程在每一个 replay 请求中都会重复，并且耗费昂贵。**Problem 6.** 第二，因为我们在分区中有我们需要跳过的冗余的消息，因此回复一个 Deliver 请求并不像设置一个 consumer，寻找请求的 offset，replay所有的记录那样简单；查询表会在任意时刻被查询，交付逻辑开始变得越来越复杂，并且对查询表的检查会增加延迟。我们先不考虑第二个问题，针对第一个问题，为了解决它，我们需要在创建 blocks 时维持它们（peersist these blocks as we create them），当我们 replaying（译者：不知道这个怎么翻译合适）时，我们只需要传输已维持着的 blocks（transmit the persisted blocks）。

**Solution 5a.** 为了解决这个问题，假设我们要全部进入Kafka，我们创建另一个**分区** - 让我们称之为分区1，这意味着我们目前为止保持引用的分区是分区0。现在，每当 OSNs 切出一个 block，它们都会将其推送 (post) 到分区1，并且所有的 Deliver 请求都将由这个分区提供服务（served by that partitioin）。因为每个 OSN 都把它们切下来的 block 推送到分区1，所以我们能想到分区1中的 block 的顺序并不是**链**中 block 的真正顺序；其中的 block 可能重复，并且由于 block 来自所有的 OSNs，因而 block 的编号不太可能严格递增 —— 如图5所示。


![Fig. 5. A possible state of partition 1. Most recent message gets appended to the right. Notice how the block numbers across all OSNs are not in a strictly increasing order (3-3-1-2-2-4). However, the numbers per OSN form a strictly increasing sequence as expected (eg. 1-2 for OSN4).](http://upload-images.jianshu.io/upload_images/7147675-c67fa17ddbcbd785.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这意味着 Kafka 分配的 offset 值并不能对应 OSN 分配的（连续的）block 编号，因此如前面所做的一样，我们在此处也需要维护一个 block-to-offset 的查询表。如表2所示。

|Block Number|Offset Number|
|:----------:|:-----------:|
|	...		 |		...	   |
|3			 |3		   |
|4			 |8		   |
|	...		 |	...		   |

以上解决方案能够工作，但注意到，我们之前所说的问题6依旧没有解决；在 OSN 侧的 Deliver 逻辑比较复杂并且 查询表 在每一次请求都需要被查询。

**Solution 6.** 想想会发现，造成这个问题的原因是“冗余的消息被推送到了分区”。无论是分区0的 TTC-X 消息（参见图4中的最后一个消息）还是“Block X”消息 它被发送到分区1并且小于或等于较早的消息（图5中的所有中间消息都小于或等于最左侧的块3）。那我们如何摆脱这些冗余消息呢？

**Solution 6a.** 首先，让我们采取如下的规则：如果你已经从**分区**收到与您要发布的内容相同的消息（减去签名），请中止你自己的推送操作。回到图5中的例子，如果 OSN1 在形成自己的 block 3 之前已经看到了分区1中的 OSN2 的 Block 3消息，则它将中止其向分区1的传输。（我们在这里描述的所有内容都具有分区0和图4示例的等效示例。）虽然这样能够去掉冗余的消息，但并不能够完全消除他们。肯定会有OSNs在同一时间发布相同的块的情况，或者相同的块正在飞往 Kafka 代理集。即使你加了一些其他操作在里面（比如，在每个节点要把 block 推送到链上时，先加一个随机的延时），你依然会面临这样的问题。

**Solution 6b.** 如果我们有一个负责推送 block 到分区1中的 OSN 领导节点，这回怎么样呢？有好几种选出 leader 的方法；例如，你可以使所有 OSN 在 ZooKeeper 中竞争 znode，或者让第一个在分区0中发布 TTC-X 消息的节点成为 leader。另一个有趣的方法是，可以让所有的 OSNs 属于同一个 consumer group，这意味着每个 OSN 在一个 topic 中都获得了自己独立的一个 partition。（假设一个 topic 负载了所有**链**的 partition-0，另一个 topic 负载了所有**链**的 partition-1）于是，负责将 TTC-X 消息发送到 partition-0或者负责把 block 推送到某条**链**的 partition-1的 OSN就是这条链的 partition 0的拥有者 —— 这是由 Kafka 管理的状态，可以通过 Kafka Metadata API 查询。（向一个不拥有有该**链/分区**的 OSN 发送的 Deliver 请求会被重定向至另外一个合适的 OSN。）

上诉方案可以工作，但是如果 leader 发送 Block X 消息给分区，然后就宕机了。这条信息还在去往分区1的路上，但还没有到达。其他 OSNs 意识到 leader 宕机了，因为 leader 不再持有 ZooKeeper 的 znode，于是一个新的 leader 被选取出来。新选出来的 leader 意识到 Block X 在自己的缓存里但还没有被推送到分区1，于是它把 Block X 推送给分区1。现在，之前在路上的那个 Block X 到达了分区1，那么问题来了，我们又再一次得到了重复的 block。这一系列事件如图6所示。


![Fig. 6. Having a leader does not eliminate duplicate messages.](http://upload-images.jianshu.io/upload_images/7147675-32d746a945350b3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Solution 6c.** 日志压缩能解决这个问题吗？日志压缩保证了 Kafka 中保存的是一个分区中每个 message key 的最新的已知 value。情景如图7所示。



![Fig. 7. Log compaction in Kafka.](http://upload-images.jianshu.io/upload_images/7147675-39ef156d2289954b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以，如果我们使用了日志压缩，我们可以完全地从分区中去除重复的部分，当然，前提是所有的 Block X 消息都携带了相同的 key，而不同的 X 对应的 key 是不同的。但是因为日志压缩存储的是一个 key 的最新版本，OSNs 可能会因为查询了过期的查询表而出错。再看到图7中的例子，假设列出的所有 key 都对应了 block，有一个仅接收该分区中的前两个消息的OSN具有将 block 1 映射到 offset 0 并且将 block 2 映射到 offset 1的查找表。与此同时，由于分区已经被压缩（如图7下半部分所示），因此查找 offset 0（或 1） 将导致错误。与日志压缩问题同样重要的是，分区1中的 blocks 并不会升序排序（如图7下半部分所示，blocks 的顺序是 1-3-4-5-2-6），所以 Deliver 逻辑还是比较复杂。实际上，考虑到查询表有过时的风险，所以日志压缩的方法不太可行。

所以这里提供的解决方案都没有解决问题6。我们来看看有没有办法解决这个问题，为此，让我们回到第一个问题5 - persisting blocks so that replays are faster（译者：不知道怎么翻译才好）。

**Solution 5b.** 相比与在 Solution 6a 中使用了另一个 Kafka 分区，在此，我们依旧使用分区0 —— transaction 和 TTC-X 消息所推送到的分区，并且每个 OSN 为每个**链**维护一个本地日志。如图8所示。



![Fig. 8. The proposed design. For simplicity we assume all transactions here belong to the same chain, but the design works for multiple chains.](http://upload-images.jianshu.io/upload_images/7147675-31edee391fa52d8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样做有如下的好处。

第一，它解决了问题6 —— **Solution 6d.**：处理一个 Deliver 请求现在只需要从本地账本里顺序读取。（不会存在重复，因为 OSN 在写本地日志时会过滤它们，也没有查询表。 OSN 会跟踪它读到的最后一个 offset 值，只是为了知道当重新连接时从 Kafka consuming 的地方）

第二，它最大限度地利用了 orderer 代码库中的通用组件的使用 - 所有现有实现中的 Deliver 路径基本上都是相同的

本地账本为 Deliver 请求提供服务的一个缺点就是，会比直接从 Kafka 提供服务慢。 但是实际上我们从来没有从 Kafka 直接提供服务； OSN上总是做了一些处理。

具体来说，如果我们讨论一下 replay 请求，替代方案（solution 4b 与 solution 5a）仍然需要在 OSNs 上进行处理（无论是 solution 4b 中的打包与对查询表的查找，还是在 solution 5a中对查询表的查找），因此在无账本解决方案（ledger-less solutions）中我们已经付出了一定的代价。

If we are talking about Deliver requests to keep current（“tail -f”等效），则解决方案6d与4b相比的额外代价是将 block 存储到账本并从其中提供。 在解决方案5a中，块需要经由分区1进行第二次往返，因此根据环境，这可能甚至比6d差。

总体而言，对传入的客户端 transaction 与 TTC-X 消息使用单个分区（每个**链**）并且将生成的 blocks 存储在本地账本（每个**链**）的排序服务在性能与复杂性之间取得了较好的平衡。
