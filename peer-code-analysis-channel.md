## 每个文件从下往上调用



### 代码结构
```
	peer/
	|-	chaincode/
	|-	channel/
	|-	clilogging/
	|-	common/
	|-	gossip/
	|-	node/
	|-	version/
	|-	main.go
	
```
---
### channel
---
##### channel/channel.go

一些结构体与变量定义
（下面这些就总结了在 channle 子文件夹下所实现的对 channel 的操作的方法）

        const (
            channelFuncName = "channel"
            shortDes        = "Operate a channel: create|fetch|join|list|update."
            longDes         = "Operate a channel: create|fetch|join|list|update."
        )

一些命令行解析

- InitCmdFactory

        type ChannelCmdFactory struct {
            EndorserClient   pb.EndorserClient
            Signer           msp.SigningIdentity
            BroadcastClient  common.BroadcastClient	//发送消息给 orderer
            DeliverClient    deliverClientIntf		//获取 block，其中有多种方法
            BroadcastFactory BroadcastClientFactory //一个函数类型
        }

    - common.GetDefaultSignerFnc()  给客户端返回一个默认的 signer。**cmdFact.Signer 接收**
    - common.GetBroadcastClientFnc 创建 BroadcastClient 接口的简单实例。**cmdFact.BroadcastFactory  接收**
    - common.GetEndorserClientFnc() 给这个 peer 返回一个新的 背书者 客户端 连接。**cmdFact.EndorserClient 接收**
    - credentials.NewClientTLSFromFile 为客户端从输入的 certificate 构建 TLS credentials
    - ab.NewAtomicBroadcastClient(xx).Deliver(xxxx) 这是干嘛的？新建广播的？
    - newDeliverClient() 返回了一个 deliverClient 结构体指针。**cmdFact.DeliverClient 接收**

最后得到 ChannelCmdFactory 类型的变量 cmdFact

---
##### channel/create.go

- **const createCmdDescription = "Create a channel"**

- createChannelFromDefaults 返回了一个 *cb.Envelope（通过函数名猜测是指 “从默认创建通道”吗）

- createChannelFromConfigTx 返回了一个 *cb.Envelope （估计是通过配置文件来“创建通道”）

- sanityCheckAndSignConfigTx
  - 做了很多数据的 unmarshal 操作
  - 通过调用 utils.CreateSignedEnvelope 返回了一个对应特定类型的 signed envelope 

- sendCreateChainTransaction
  - 执行通道创建
  - 调用 sanityCheckAndSignConfigTx，返回 chCrtEnv
  - 新建了一个 common.BroadcastClient 接口的变量，调用 Send 方法，发送了上面得到的 chCrtEnv

- executeCreate
  - 调用 sendCreateChainTransaction
  - 调用 getGenesisBlock 获取创世区块
  - proto.Marshal 对所获的块解压
  - 调用 ioutil.WriteFile 写了一个 chainID.block 的区块（不过 chainID 从何而来没找到，估计是个全局变量）

- create 
  - 如果传入参数中的 cf *ChannelCmdFactory 为空，则调用 /channel/channel.go 的 InitCmdFactory 函数
  - 调用 executeCreate(cf)

---
##### channel/deliverclient.go

- 一些接口与结构体定义

        type deliverClientIntf interface {
            getSpecifiedBlock(num uint64) (*common.Block, error)
            getOldestBlock() (*common.Block, error)
            getNewestBlock() (*common.Block, error)
            Close() error
        }


        type deliverClient struct {
            conn    *grpc.ClientConn 					//代表一个客户端向 gRPC 的链接
            client  ab.AtomicBroadcast_DeliverClient    //有发送与接收两个方法
            chainID string
        }

- newDeliverClient 返回一个对  deliverClient 结构体的引用

- seekHelper 同样通过调用 utils.CreateSignedEnvelope 返回了一个对应特定类型的 signed envelope 

- r *deliverClient 的方法
  - seekSpecified 调用 r.client.Send(seekHelper())，需要传入参数 blockNumber
  - seekOldest 无需参数
  - seekNewest 无需参数
  - readBlock 读取 block
  - getSpecifiedBlock 调用 seekSpecified 与readBlock
  - getOldestBlock 调用 seekOldest 与 readBlock
  - getNewestBlock 调用 seekNewest 与 readBlcok
  - Close 调用 r.conn.Close()


各种查找方法发送的参数是 chainID 与 SeekPosition
​    
- getGenesisBlock select 语句，没太看懂这个 select 语言

---
##### channnel/fetchconfig.go

- fetchCmd 命令行解析

- fetch
  - 如果传入的 cf *ChannelCmdFactory 为空，则调用 InitCmdFactory
  - 判断参数 args[0]（命令行的输入），根据参数取 block
    - "oldest" 则执行 cf.DeliverClient.getOldestBlock()
    - "newest"则执行 cf.DeliverClient.getNewestBlock()
    - "config" 则执行 iBlock = cf.DeliverClient.getNewestBlock()，lc = utils.GetLastConfigIndexFromBlock(iBlock)，block = cf.DeliverClient.getSpecifiedBlock(lc)
    - 默认则执行 cf.DeliverClient.getSpecifiedBlock(uint64(num))

  - 执行proto.Marshal(block)，压缩 block 
  - 调用 ioutil.WriteFile 写文件

---
##### channel/join.go

- **const commandDescription = "Joins the peer to a chain."**

- joinCmd 命令行解析

- getJoinCCSpec 输入参数为 *pb.ChaincodeSpec，该参数携带了 chaincode 的明确（specification），ChaincodeSpec 结构体中的字段就是定义一个 chaincode 所需的实际元数据
  - ioutil.ReadFile(genesisBlockPath) 读取创世区块
  - 执行 spec := &pb.ChaincodeSpec{xxx}，得到对 chaincode 的 CCSpec
  - 返回 spec

- executeJoin
  - 执行 spec, err := getJoinCCSpec()
  - 执行 invocation := &pb.ChaincodeInvocationSpec{ChaincodeSpec: spec}，得到 ChaincodeInvocationSpec 结构体的引用，该结构体携带了 chaincode 的函数和参数
  - 调用 putils.CreateProposalFromCIS 通过给定的序列化的实体与 ChaincodeInvocationSpec 结构体，返回了一个 proposal
  - 调用 putils.GetSignedProposal 返回了一个 *pb.SignedProposal 的 signedProp，SignedProposal 有两个字段，一个是 ProposalBytes，另一个是 Signature
  - 调用 cf.EndorserClient.ProcessProposal 返回了一个 *pb.ProposalResponse 指针，ProposalResponse 结构体是从一个 endorser 返回给 proposal submitter 的

- join
  - 一些命令行处理
  - InitCmdFactory
  - 执行 executeJoin

---
##### channel/list.go

- 定义一个 endorserClient 的结构体

- listCmd 命令行解析

- getChannels，endorserClient 的方法，输入是 ChannelInfo 的结构体，其实就是一个  ChannelId
  - 执行 invocation := &pb.ChaincodeInvocationSpec（注意一下 ChaincodeId 是 cscc，Input 是 GetChannels
  - 执行 cc.cf.Signer.Serialize()
  - 调用 putils.CreateProposalFromCIS 通过给定的序列化的实体与 ChaincodeInvocationSpec 结构体，返回了一个 proposal
  - 调用 utils.GetSignedProposal 返回一个 signed proposal
  - 调用 cc.cf.EndorserClient.ProcessProposal 得到 proposalResp
  - 调用 proto.Unmarshal，将之前得到的 proposalResp 解压为一个 pb.ChannelQueryResponse 变量 channelQueryResponse
  - 返回 channelQueryResponse.Channels，其实就是一个 channelID

- list
  - 调用 InitCmdFactory
  - 执行 client := &endorserClient{cf}
  - 调用 client.getChannels()，在 log 中输出相关信息，如 `"Channels peers has joined to: %s", channel.ChannelId`

---
##### channel/signconfigtx.go

- signconfigtxCmd 命令行解析

- sign
  - 判断 channelTxFile 即 configtx 文件路径是否为空
  - 调用 InitCmdFactory
  - 调用 ioutil.ReadFile(channelTxFile) 读取文件
  - 调用 utils.UnmarshalEnvelope(fileData) 解压上一步读取的数据
  - 调用 channel.go 下的函数 sanityCheckAndSignConfigTx，输入参数为上一步的输出
  - 调用 utils.MarshalOrPanic 输入参数为上一步输出
  - 调用 ioutil.WriteFile 写文件，输入参数为 channelTxFile 与上一步的输出

---
##### channel/update.go

- updateCmd 命令行解析

- update
  - 判断 chainID 是否为空
  - 判断 channelTxFile 即 configtx 文件路径是否为空
  - 调用 InitCmdFactory
  - 调用 ioutil.ReadFile(channelTxFile) 读取文件
  - 调用 utils.UnmarshalEnvelope(fileData) 解压上一步读取的数据
  - 调用 channel.go 下的函数 sanityCheckAndSignConfigTx，输入参数为上一步的输出，输出为 sCtxEnv *cb.Envelope
  - 声明 common.BroadcastClient 类型变量 broadcastClient，执行broadcastClient.Send(sCtxEnv)


---
### node
---
##### node/start.go

- initSysCCs 开启 chaincodes
  - DeploySysCCs 部署系统 chaincodes
    - 部署了许多个 SysCCs，包括 cscc，lscc，escc，vscc，gscc，rscc
    - SysCCs 的结构详见 SystemChaincode 结构体
    - buildSysCC 来建立 chaincode

- server
  - txprocessors 维护的是客户交易类型与对应交易处理器的map
  - ledgermgmt.Initialize 初始化 ledgermgmt
  - peer.GetPeerEndpoint 从缓存的配置中返回 peerEndpoint （peerEndpoint 是什么呢？）
  - peer.GetSecureConfig() 返回了peer 的安全服务器配置（secure server configuration）
  - peer.CreatePeerServer 创建了一个 comm.GRPCServer 实例，这个实例是用来 peer communications （指peer 间？还是 peer 接收其他信息的操作的）
  - ccprovider.EnableCCInfoCache() 启用chaincode info 的缓存
  - accesscontrol.NewCA() 为 chaincode 服务创建一个自签名的 CA
  - createChaincodeServer 创建一个 chaincode 监听者
  - pb.RegisterAdminServer 注册管理服务器
  - endorser.NewEndorserServer 创建并返回一个新的背书服务器实例
  - pb.RegisterEndorserServer 注册背书服务器
  - viper.GetStringSlice 初始化 gossip 组件
  - service.InitGossipService 初始化 gossip 服务
  - initSysCCs()
  - peer.Initialize 初始化 peer 所有的链，该函数应该在 ledger 和 gossip 准备好后调用
  - peerServer.Start() 启动 grpc server
  - ehubGrpcServer.Start() 启动 event hub server（这是什么）
---
##### node/node.go

没什么东西
---
##### node/status.go

- status() 获取服务器状态










































