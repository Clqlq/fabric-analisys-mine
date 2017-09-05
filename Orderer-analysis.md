目前不解的问题：
* producer 是谁？干嘛的？
* consumer 是谁？干嘛的？
* tx 从被客户端提交到block 生成发生了什么？
* 客户端、orderer、 kafka集群之间的通信？


## README.md 文档
### 协议定义
在fabric中有两个原子广播排序协议，由`hyperledger/fabric/protos/orderer/ab.proto`文件描述（***我需要去学一下 proto 的相关内容了***）
### 服务类型
* Solo 
* Kafka-based
* PBFT(pending)

### 账本类型
* File ledger(production): 基于文件的账本直接将 blocks 存在文件系统里。 block 的位置通过一个轻量级的 LevelDB 来通过其编号来索引。在生产环境中，默认并推荐这种账本类型。
* RAM ledger(testing): ...
* JOSN ledger(testing): 这种文件账本的实现是一种简单的面向开发的账本，将批处理作为JSON编码文件存储在文件系统上。 这样做的目的是使检查分类帐容易，并允许CFT。 这个账本并不是为了良好的性能，而是要简单易于部署和理解。

#### 选择一个账本类型
可以在运行`orderer`二进制文件之前，设置环境变量`ORDERER_GENERAL_LEDGERTYPE`，接受三种值`file` (default), `ram`, `json`

### 尝试排序服务
为了试验排序服务，你只需在`hyperledger/fabric/orderer`目录下运行`go build`命令。然后，你可以调用没有参数的 orderer 二进制文件，也可以通过分别设置环境变量`ORDERER_GENERAL_LISTENADDRESS`，`ORDERER_GENERAL_ LISTENPORT`和`ORDERER_GENERAL_LEDGER_TYPE`来覆盖绑定地址，端口和账本类型。

#### 剖析
可以使用orderer.yaml文件或环境变量来配置分析服务。 要启用分析设置ORDERER_GENERAL_PROFILE_ENABLED = true，并且可选地将ORDERER_GENERAL_PROFILE_ADDRESS设置为分析服务所需的网络地址。 默认地址为0.0.0.0:6060，如Golang文档。


## orderer.yaml 文档

里面的内容不是很明确
















