代码结构
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
### node
---
 node/start.go

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
node/node.go

没什么东西
---
node/status.go

- status() 获取服务器状态

---
### channel
---
channel/channel.go

一些结构体与变量定义
一些命令行解析

- InitCmdFactory
	- common.GetDefaultSignerFnc()  给客户端返回一个默认的 signer。**cmdFact.Signer 接收**
	- common.GetBroadcastClientFnc 创建 BroadcastClient 接口的简单实例。**cmdFact.BroadcastFactory  接收**
	- common.GetEndorserClientFnc() 给这个 peer 返回一个新的 背书者 客户端 连接。**cmdFact.EndorserClient 接收**
	- credentials.NewClientTLSFromFile 为客户端从输入的 certificate 构建 TLS credentials
	- ab.NewAtomicBroadcastClient(xx).Deliver(xxxx) 这是干嘛的？新建广播的？
	- newDeliverClient() 返回了一个 deliverClient 结构体指针。**cmdFact.DeliverClient 接收**

---
channel/create.go

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
channel/deliverclient.go

- 一些接口与结构体定义

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
	
- getGenesisBlock select 语句，每太看懂这个 select 语言的 case 部分





























