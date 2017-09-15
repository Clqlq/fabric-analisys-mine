## 每个文件从下往上调用着看



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
### chaincode
---
##### chaincode/chaincode.go

一些结构体与变量定义
（下面这些就总结了在 chaincode 子文件夹下所实现的对 chaincode 的操作的方法）

        const (
            chainFuncName = "chaincode"
            shortDes      = "Operate a chaincode: install|instantiate|invoke|package|query|signpackage|upgrade|list."
            longDes       = "Operate a chaincode: install|instantiate|invoke|package|query|signpackage|upgrade|list."
        )

一些命令行解析

一些与 chaincode 相关的变量

        var (
            chaincodeLang     string
            chaincodeCtorJSON string
            chaincodePath     string
            chaincodeName     string
            chaincodeUsr      string // Not used
            chaincodeQueryRaw bool
            chaincodeQueryHex bool
            customIDGenAlg    string
            chainID           string
            chaincodeVersion  string
            policy            string
            escc              string
            vscc              string
            policyMarhsalled  []byte
            orderingEndpoint  string
            tls               bool
            caFile            string
            transient         string
        )

剩下的没看出什么来

---
##### chaincode/common.go
**这个文件下定义了一些通用函数，提供给当前目录下的其他文件中的调用，比如 install instantiate invoke 等**

- 下面先列举在此部分代码中常用的结构体

		type ChaincodeSpec struct {
			Type        ChaincodeSpec_Type `protobuf:"varint,1,opt,name=type,enum=protos.ChaincodeSpec_Type" json:"type,omitempty"`
			ChaincodeId *ChaincodeID       `protobuf:"bytes,2,opt,name=chaincode_id,json=chaincodeId" json:"chaincode_id,omitempty"`
			Input       *ChaincodeInput    `protobuf:"bytes,3,opt,name=input" json:"input,omitempty"`
			Timeout     int32              `protobuf:"varint,4,opt,name=timeout" json:"timeout,omitempty"`
		}

---
		type ChaincodeDeploymentSpec struct {
			ChaincodeSpec *ChaincodeSpec `protobuf:"bytes,1,opt,name=chaincode_spec,json=chaincodeSpec" json:"chaincode_spec,omitempty"`
			// Controls when the chaincode becomes executable.
			EffectiveDate *google_protobuf1.Timestamp                  `protobuf:"bytes,2,opt,name=effective_date,json=effectiveDate" json:"effective_date,omitempty"`
			CodePackage   []byte                                       `protobuf:"bytes,3,opt,name=code_package,json=codePackage,proto3" json:"code_package,omitempty"`
			ExecEnv       ChaincodeDeploymentSpec_ExecutionEnvironment `protobuf:"varint,4,opt,name=exec_env,json=execEnv,enum=protos.ChaincodeDeploymentSpec_ExecutionEnvironment" json:"exec_env,omitempty"`
		}
---
		type Proposal struct {
			// The header of the proposal. It is the bytes of the Header
			Header []byte `protobuf:"bytes,1,opt,name=header,proto3" json:"header,omitempty"`
			// The payload of the proposal as defined by the type in the proposal
			// header.
			Payload []byte `protobuf:"bytes,2,opt,name=payload,proto3" json:"payload,omitempty"`
			// Optional extensions to the proposal. Its content depends on the Header's
			// type field.  For the type CHAINCODE, it might be the bytes of a
			// ChaincodeAction message.
			Extension []byte `protobuf:"bytes,3,opt,name=extension,proto3" json:"extension,omitempty"`
		}

---
		type SignedProposal struct {
			// The bytes of Proposal
			ProposalBytes []byte `protobuf:"bytes,1,opt,name=proposal_bytes,json=proposalBytes,proto3" json:"proposal_bytes,omitempty"`
			// Signaure over proposalBytes; this signature is to be verified against
			// the creator identity contained in the header of the Proposal message
			// marshaled as proposalBytes
			Signature []byte `protobuf:"bytes,2,opt,name=signature,proto3" json:"signature,omitempty"`
		}
---
		type ProposalResponse struct {
			// Version indicates message protocol version
			Version int32 `protobuf:"varint,1,opt,name=version" json:"version,omitempty"`
			// Timestamp is the time that the message
			// was created as  defined by the sender
			Timestamp *google_protobuf1.Timestamp `protobuf:"bytes,2,opt,name=timestamp" json:"timestamp,omitempty"`
			// A response message indicating whether the
			// endorsement of the action was successful
			Response *Response `protobuf:"bytes,4,opt,name=response" json:"response,omitempty"`
			// The payload of response. It is the bytes of ProposalResponsePayload
			Payload []byte `protobuf:"bytes,5,opt,name=payload,proto3" json:"payload,omitempty"`
			// The endorsement of the proposal, basically
			// the endorser's signature over the payload
			Endorsement *Endorsement `protobuf:"bytes,6,opt,name=endorsement" json:"endorsement,omitempty"`
		}







- `func checkSpec(spec *pb.ChaincodeSpec) error{}` 对输入的 spec *pb.ChaincodeSpec 进行检查
	- 如果 spec 为空，返回报错
	- 若不为空，则执行 platforms.Find(spec.Type)，根据 spec 的ChaincodeSpec_Type 返回特定平台的接口，ChaincodeSpec_Type 有如下类型：

			   const (
			       ChaincodeSpec_UNDEFINED ChaincodeSpec_Type = 0
			       ChaincodeSpec_GOLANG    ChaincodeSpec_Type = 1 
			       ChaincodeSpec_NODE      ChaincodeSpec_Type = 2
			       ChaincodeSpec_CAR       ChaincodeSpec_Type = 3
			       ChaincodeSpec_JAVA      ChaincodeSpec_Type = 4
			   )
	
	- 执行 platform.ValidateSpec(spec)，每一个平台都实现了对应的 ValidateSpec 方法，调用相关 ValidateSpec 方法，验证相关 chaincode，主要是检查给定路径的 chaincode 是否存在
	
- `func getChaincodeDeploymentSpec(spec *pb.ChaincodeSpec, crtPkg bool) (*pb.ChaincodeDeploymentSpec, error) {}`
	- 如果当前不是 development-mode 并且 crtPkg 为 True，则执行
		- 调用 checkSpec 对 spec 进行检查
		- 调用 container.GetChaincodePackageBytes，使用提供的链码规范为docker容器生成创建字，即 codePackageBytes
	- 执行 `chaincodeDeploymentSpec := &pb.ChaincodeDeploymentSpec{ChaincodeSpec: spec, CodePackage: codePackageBytes}`拼接出 chaincodeDeploymentSpec
	- 返回 chaincodeDeploymentSpec
	

- `func getChaincodeSpec(cmd *cobra.Command) (*pb.ChaincodeSpec, error) {}` 从 cli cmd 参数获取链码规范，即将命令行参数拼接成链码规范，最后返回链码规范
	- 首先执行 checkChaincodeCmdParams(cmd) 进行检查
	- 对 chaincode/chaincode.go 中的一些全局参数做处理，例如解压，装换为大写等，不赘述
	- 拼接出链码规范 spec 
	
			spec = &pb.ChaincodeSpec{
				Type:        pb.ChaincodeSpec_Type(pb.ChaincodeSpec_Type_value[chaincodeLang]),
				ChaincodeId: &pb.ChaincodeID{Path: chaincodePath, Name: chaincodeName, Version: chaincodeVersion},
				Input:       input,
			}


	- 返回 spec



- `func chaincodeInvokeOrQuery(cmd *cobra.Command, args []string, invoke bool, cf *ChaincodeCmdFactory) (err error) {}`
	- 调用 getChaincodeSpec(cmd)，拿到链码规范 spec
	- 调用 ChaincodeInvokeOrQuery(spec, chainID, invoke, cf.Signer, cf.EndorserClient, cf.BroadcastClient) 




		- chaincodeInvokeOrQuery
		  - 调用 getChaincodeSpec(cmd) 得到 spec
		  - 调用 ChaincodeInvokeOrQuery 得到 proposalResp  *pb.ProposalResponse
		  - 判断 invoke
		    - 如果是对 chaincode 执行调用，则对上一步返回的 proposalResp 中的相关字段进行解压与判断，打印相关信息
		    - 如果是对 chaincode 执行查询，则对 chaincodeQueryRaw 与 chaincodeQueryHex 两个布尔变量执行判断，选择相关的格式打印查询结果

- ChaincodeCmdFactory 结构体 holds the clients used by ChaincodeCmd

   type ChaincodeCmdFactory struct {
   		EndorserClient  pb.EndorserClient
   		Signer          msp.SigningIdentity
   		BroadcastClient common.BroadcastClient
   	}

- InitCmdFactory 使用默认的 clients 初始化 ChaincodeCmdFactory
  - endorserClient, err = common.GetEndorserClientFnc()
  - signer, err := common.GetDefaultSignerFnc()
  - broadcastClient, err = common.GetBroadcastClientFnc(orderingEndpoint, tls, caFile)


- `func ChaincodeInvokeOrQuery(spec *pb.ChaincodeSpec, cID string, invoke bool, signer msp.SigningIdentity, endorserClient pb.EndorserClient, bc common.BroadcastClient,) (*pb.ProposalResponse, error) {}` 
	- 对全局以及局部的参数做一些处理
	- 调用 putils.CreateChaincodeProposalWithTransient，该函数通过给定的输入创建了一个 proposal，对该 proposal 填充了Header 与 Payload 两个字段（Header 与 Payload 字段的具体内容见其定义）
	- 调用 putils.GetSignedProposal(prop, signer)，通过给定的 proposal message 与 signing identity 返回了一个signedProp *pb.SignedProposal
	- 调用 endorserClient.ProcessProposal(context.Background(), signedProp) 
		- `func (c *endorserClient) ProcessProposal(ctx context.Context, in *SignedProposal, opts ...grpc.CallOption) (*ProposalResponse, error) {} `方法发送 gRPC 消息，将 SignedProp 发送给 Endorser，并返回 proposalResp *pb.ProposalResponse
	- 调用 putils.CreateSignedTx 将 proposal signer ProResp 构建成一个 env *commom.Envelop
	- 调用 bc.Send(env)





			- ChaincodeInvokeOrQuery
			  - 调用 putils.CreateChaincodeProposalWithTransient，该函数通过给定的输入创建了一个 proposal，返回的内容里除了 proposal 还有该 proposal 的 transaction id，返回 prop *peer.Proposal
			  - 调用 putils.GetSignedProposal(prop, signer)，通过给定的 proposal message 与 signing identity 返回了一个 signed proposal，即 signedProp *pb.SignedProposal
			  - 调用 endorserClient.ProcessProposal 返回 proposalResp *pb.ProposalResponse
			  - 判断 invoke
			    -如果为真，则调用 putils.CreateSignedTx，将 proposal, endorsements, signer 压缩到 env *common.Envelope，调用 bc.Send(env)，最后返回 proposalResp
			    - 为假，则直接返回 proposalResp

---
##### chaincode/install.go

		const installDesc = "Package the specified chaincode into a deployment spec and save it on the peer's path."

命令行解析

- install 将 depspec 安装到 peer.address，输入参数为 msg proto.Message, cf *ChaincodeCmdFactory 
  - 调用 utils.CreateInstallProposalFromCDS，返回 prop *peer.Proposal
  - 调用 utils.GetSignedProposal，对 proposal 签名 
  - 调用 cf.EndorserClient.ProcessProposal 对签名后的 signedproposal 拼上 response，得到 proposalResponse

- genChaincodeDeploymentSpec creates ChaincodeDeploymentSpec as the package to install （package 怎么理解？什么概念？）
	- 调用 getChaincodeSpec(cmd)，返回 spec *pb.ChaincodeSpec
	- 调用 getChaincodeDeploymentSpec，返回 csd *pb.ChaincodeDeploymentSpec
	- 返回 csd
	
- getPackageFromFile 从文件获取 chaincode package 并解压出 ChaincodeDeploymentSpec，输入参数为 ccpackfile string
	- 执行 ioutil.ReadFile(ccpackfile)，读取文件
	- 调用 ccprovider.GetCCPackage，该函数











































