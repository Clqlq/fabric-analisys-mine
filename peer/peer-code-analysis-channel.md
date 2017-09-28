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
##### channel/channel.go

一些结构体与变量定义
（下面这些就总结了在 channle 子文件夹下所实现的对 channel 的操作的方法）

        const (
            channelFuncName = "channel"
            shortDes        = "Operate a channel: create|fetch|join|list|update."
            longDes         = "Operate a channel: create|fetch|join|list|update."
        )

一些命令行解析

一些与 channel 相关的变量
```go
	type OrdererRequirement bool
	type EndorserRequirement bool
```
```go
	const (
		EndorserRequired    EndorserRequirement = true
		EndorserNotRequired EndorserRequirement = false
		OrdererRequired     OrdererRequirement  = true
		OrdererNotRequired  OrdererRequirement  = false
	)
```
```go
	var (
		// join related variables.
		genesisBlockPath string

		// create related variables
		chainID                    string
		channelTxFile              string
		orderingEndpoint           string
		tls                        bool
		caFile                     string
		ordererTLSHostnameOverride string
		timeout                    int
	)

```


- InitCmdFactory
	- common.GetDefaultSignerFnc()  给客户端返回一个默认的 signer。**cmdFact.Signer 接收**
	- common.GetBroadcastClientFnc 创建 BroadcastClient 接口的简单实例。**cmdFact.BroadcastFactory  接收**
	- common.GetEndorserClientFnc() 给这个 peer 返回一个新的 背书者 客户端 连接。**cmdFact.EndorserClient 接收**
	- credentials.NewClientTLSFromFile 为客户端从输入的 certificate 构建 TLS credentials
	- ab.NewAtomicBroadcastClient(xx).Deliver(xxxx) 这是干嘛的？新建广播的？
  	- newDeliverClient() 返回了一个 deliverClient 结构体指针。**cmdFact.DeliverClient 接收**
	- 最后得到 ChannelCmdFactory 类型的变量 cmdFact

剩下的也没什么了

---
##### channel/create.go
```
	const createCmdDescription = "Create a channel"
```
### 生成通道的过程是不需要背书的

- `func create(cmd *cobra.Command, args []string, cf *ChannelCmdFactory) error {}` 该文件下的调用各个函数的函数
	- 执行 InitCmdFactory(EndorserNotRequired, OrdererRequired)
	- 执行 executeCreate(cf)

- `func executeCreate(cf *ChannelCmdFactory) error {}` 该文件下的主要函数
	- 执行 sendCreateChainTransaction(cf) 发送新建链的交易
	- 执行 getGenesisBlock(cf) 获取创世区块
	- 执行 proto.Marshal(block)，将创世区块解压
	- `file := chainID + ".block"`，执行 ioutil.WriteFile(file, b, 0644)，写文件

- `func sendCreateChainTransaction(cf *ChannelCmdFactory) error {}`
	- 通过判断 channelTxFile 决定是执行 createChannelFromConfigTx(channelTxFile) 还是执行 createChannelFromDefaults(cf)，目的都是生成一个创建通道的交易并打包成 envelope 类型
	- 执行 sanityCheckAndSignConfigTx(chCrtEnv) 主要是在输入的 chCrtEnv 上附加上签名，并重新打包成 envelope 格式
	- 执行 broadcastClient.Send(chCrtEnv) 将 chCrtEnv 发给 orderer
	- 执行 broadcastClient.Close()

- `func createChannelFromConfigTx(configTxFileName string) (*cb.Envelope, error) {}` 从配置文件中生成新建 channel 的交易
	- 执行 ioutil.ReadFile(configTxFileName) 从配置文件中读取配置
	- 执行 utils.UnmarshalEnvelope(cftx) 将读取到的数据解压


- `func createChannelFromDefaults(cf *ChannelCmdFactory) (*cb.Envelope, error) {}` 从默认中生成新建 channel 的交易
	- 执行 mspmgmt.GetLocalMSP().GetDefaultSigningIdentity() 获取签名者
	- 执行 channelconfig.MakeChainCreationTransaction(chainID, genesisconfig.SampleConsortiumName, signer) 将默认的一些配置拼接成 chCrtEnv *cb.Envelop 

- `func sanityCheckAndSignConfigTx(envConfigUpdate *cb.Envelope) (*cb.Envelope, error) {}` 主要就是将接收到的 envelop 信息解压，核对相关信息，附加本地签名者的签名（与之前的签名区别是不是就在于之前的签名是 org 里面的 Admin 签的，这儿的签名是发起建立通道的 peer 签的？），再将信息打包成 envelop

---
##### channel/deliverclient.go

- 一些接口与结构体定义

```go
type deliverClientIntf interface {
	getSpecifiedBlock(num uint64) (*common.Block, error)
	getOldestBlock() (*common.Block, error)
	getNewestBlock() (*common.Block, error)
	Close() error
}
```
```go
type deliverClient struct {
	conn    *grpc.ClientConn
	client  ab.AtomicBroadcast_DeliverClient
	chainID string
}
```
- 该文件主要是给 deliverClient 定义了一系列的方法，用来进行各种取块的操作，核心是下面几个方法：
```go
	- func (r *deliverClient) seekSpecified(blockNumber uint64) error {}
	- func (r *deliverClient) seekOldest() error {}
	- func (r *deliverClient) seekNewest() error {}
	- func (r *deliverClient) readBlock() (*common.Block, error) {}
```
在上述方法的基础上又定义了几个获取 block 的方法：
```go
	- func (r *deliverClient) getSpecifiedBlock(num uint64) (*common.Block, error) {}
	- func (r *deliverClient) getOldestBlock() (*common.Block, error) {}
	- func (r *deliverClient) getNewestBlock() (*common.Block, error) {
```	
此外还定义了 `func getGenesisBlock(cf *ChannelCmdFactory) (*common.Block, error) {}`的方法

- 总结来看，该文件下定义的方法核心都是先调用`func seekHelper(chainID string, position *ab.SeekPosition) *common.Envelope {}`函数，将需要要发出的查询信息打包，然后再调用 r.client.Send() 将查询信息发送到 orderer

---
##### channel/fetchconfig.go

### 获取配置无需背书节点，需要排序节点

- 一些命令行解析

- `func fetch(cmd *cobra.Command, args []string, cf *ChannelCmdFactory) error {}`该文件下的主要函数，通过调用 deliverclient.go 下的方法来获取块或者通道的配置
	- 执行 InitCmdFactory(EndorserNotRequired, OrdererRequired)
	- 根据传入的命令参数选择采取下述动作：
		- cf.DeliverClient.getOldestBlock()
		- cf.DeliverClient.getNewestBlock()
		- 从 Block 中获取最新的配置
			- 执行 utils.GetLastConfigIndexFromBlock(iBlock) 获取最新的配置块的编号
			- 执行 cf.DeliverClient.getSpecifiedBlock(lc) 取得配置块

	- 执行 proto.Marshal(block) 将取得的块压缩
	- 执行 ioutil.WriteFile(file, b, 0644) 将压缩后的内容写入文件

---
##### channel/join.go

### 加入通道需要背书节点，无需排序节点

```go
		const commandDescription = "Joins the peer to a chain."
```

- `func join(cmd *cobra.Command, args []string, cf *ChannelCmdFactory) error {}` 该文件下的主要函数
	- 执行  InitCmdFactory(EndorserRequired, OrdererNotRequired)
	- 执行 executeJoin(cf)

- `func executeJoin(cf *ChannelCmdFactory) (err error) {}` 
	- 调用 getJoinCCSpec() 获取返回的链码规范
	- 后续就和之前提到过的很多对链码的操作一样，构造一个提案，并将提案发送给 Endorser 并获取对提案的回复
	- 如果 proposalResp 正确，则表示 peer 加入通道成功

---
##### channel/list.go

### 列出通道需要背书节点，无需排序节点

- `func list(cf *ChannelCmdFactory) error {}` 该文件下的主要函数
	- 执行 InitCmdFactory(EndorserRequired, OrdererNotRequired)
	- 执行 client := &endorserClient{cf} 新建一个 endorserClient，endorserClient 这个结构体里只有 cf *ChannelCmdFacory 这个字段
	- 执行 client.getChannels() 打印出 peers 已经加入过的 channels
	
- `func (cc *endorserClient) getChannels() ([]*pb.ChannelInfo, error) {}`
	- 和前面提到过的过程基本一致，即构造一个 proposal，发送给 endorser，获取在 endoerser 的回复
###（发现其实所有的 peer 操作的核心都是 `cc.cf.EndorserClient.ProcessProposal` 这个函数的实现，它如何怎么处理各种提案即为后续学习的重点）

---
##### channel/signconfig.go

### 无需背书节点，无需排序节点

- `func sign(cmd *cobra.Command, args []string, cf *ChannelCmdFactory) error {}` 对配置交易进行签名
	- 执行 InitCmdFactory(EndorserNotRequired, OrdererNotRequired)
	- 执行 ioutil.ReadFile(channelTxFile) 读取通道配置文件
	- 执行 sanityCheckAndSignConfigTx(ctxEnv) 加上签名
	- 执行 utils.MarshalOrPanic(sCtxEnv) 压缩 envelop 
	- 执行 ioutil.WriteFile(channelTxFile, sCtxEnvData, 0660) 写文件

---
##### channel/update.go

### 无需背书节点，需要排序节点

- `func update(cmd *cobra.Command, args []string, cf *ChannelCmdFactory) error {}`
	- 提供 channelID 与 通道配置文件
	- 执行 InitCmdFactory(EndorserNotRequired, OrdererRequired)
	- 执行 ioutil.ReadFile(channelTxFile) 读取通道配置文件
	- 执行 utils.UnmarshalEnvelope(fileData) 解压数据
	- 执行 sanityCheckAndSignConfigTx(ctxEnv) 加上签名
	- 执行 broadcastClient.Send(sCtxEnv) 将 envelop 发送给 orderer
	- 执行 broadcastClient.Close()
	



































