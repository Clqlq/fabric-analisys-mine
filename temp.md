common.go

		type ChaincodeCmdFactory struct {
			EndorserClient  pb.EndorserClient
			Signer          msp.SigningIdentity
			BroadcastClient common.BroadcastClient
		}


- `func chaincodeInvokeOrQuery(cmd *cobra.Command, args []string, invoke bool, cf *ChaincodeCmdFactory) (err error) {}` chaincode 调用或查询
	- spec 部分
	- 调用 ChaincodeInvokeOrQuery 完成对 chaincode 的签名、背书，获得 proposalResp *ProposalResponse
	- 对 invoke 进行判断
		- invoke 为真，对 proposalResp.Response.Status 进行判断
			- 若 proposalResp.Response.Status >= shim.ERROR 为真， 则进一步判断是哪部分造成了失败
			- 若 proposalResp.Response.Status >= shim.ERROR 为假，则验证是否能够解析出 proposalResp 的 proposalResp.Payload，proposalResp.Payload.Extension，若均能正确解析则打印 chaincode invoke successful 的日志
		- 若 invoke 为假，则执行 query 操作
			- 对变量 chaincodeQueryRaw 与 chaincodeQueryHex 进行判断，执行相关的输出操作

- `func checkChaincodeCmdParams(cmd *cobra.Command) error {}`  对 chaincode 的 cmdParams 执行检查
	- 检查 chaincodeName 是否为空
	- 检查 chaincodeVersion 是否为空
	- 检查是否已定义 escc vscc
	- 检查 policy
	- 检查 chaincodeCtorJSON 参数设置是否正确


- `func InitCmdFactory(isEndorserRequired, isOrdererRequired bool) (*ChaincodeCmdFactory, error) {}`使用默认的 clients 初始化 ChaincodeCmdFactory
	- 若 isEndorserRequired 为真，即需要背书节点，则调用 common.GetEndorserClientFnc() 返回一个默认背书 client 连接
		- 调用 peer.NewPeerClientConnection() 返回一个 clientConn *grpc.ClientConn
		- 调用 pb.NewEndorserClient(clientConn) 将 clientConn 存储在结构体 endorserClient 中
		- 返回 endorserClient（所以endorserClient 是一个 gRPC 连接）
	- 调用 common.GetDefaultSignerFnc() 返回一个默认 signer
		- 调用 mspmgmt.GetLocalMSP().GetDefaultSigningIdentity() 返回 signer msp.SigningIdentity
	- 若 isOrdererRequired 为真，即需要排序节点
		- 若给定的 orderingEndpoint 为空，调用 common.GetOrdererEndpointOfChainFnc(chainID, signer, endorserClient)，获得一组排序端点字符串组，选择第一个为 orderingEndpoint （**也就是说每条链会选择一个 orderer 作为排序服务的端点？**）
	- 调用 common.GetBroadcastClientFnc(orderingEndpoint, tls, caFile)创建并返回一个 BroadcastClient 接口的简单实例
	- 将前面得到的 endorserClient，signer，broadcastClient，拼接成结构体 ChaincodeCmdFactory 并返回对其的引用

------------------
------------------

install.go

install.go 中的主要函数就是`func chaincodeInstall(cmd *cobra.Command, ccpackfile string, cf *ChaincodeCmdFactory) error {}`

- `func chaincodeInstall(cmd *cobra.Command, ccpackfile string, cf *ChaincodeCmdFactory) error {}`
	- 先执行 InitCmdFactory(true, false)，得到 cf ChaincodeCmdFactory，注意此时没有 ordererendpoint，即 cf 中的 BroadcastClient 字段是空的
	- 对 ccpackfile 进行判断
		- 若为空则调用 genChaincodeDeploymentSpec(cmd, chaincodeName, chaincodeVersion)，返回一个 ccpackmsg ChaincodeDeploymentSpec
		- 若不为空则调用 getPackageFromFile(ccpackfile) 从文件中获取链码部署说明
	- 调用 install(ccpackmsg, cf) 执行安装，将链码部署说明安装到特定 peer.address 上
	
- `func install(msg proto.Message, cf *ChaincodeCmdFactory) error {}`
	- 调用 utils.CreateInstallProposalFromCDS(msg, creator)从传入的参数生成一个 proposal
	- 对 proposal 签名，生成 signedProp
	- cf.EndorserClient.ProcessProposal，将 signedProp 发给 Endorser 并获取 proposalResponse

- 其他函数