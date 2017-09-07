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
node/start.go

- initSysCCs 开启 chaincodes
	- DeploySysCCs 部署系统 chaincodes
		- 部署了许多个 SysCCs，包括 cscc，lscc，escc，vscc，gscc，rscc
		- SysCCs 的结构详见 SystemChaincode 结构体
		- buildSysCC 来建立 chaincode






















