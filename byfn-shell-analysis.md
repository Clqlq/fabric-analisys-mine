## 1. 环境变量配置
```bash
export PATH=${PWD}/../bin:${PWD}:$PATH
export FABRIC_CFG_PATH=${PWD}
```
设置了临时环境变量，使用export命令行声明即可，变量在关闭shell时失效。

## 2. `./byfn.sh -m generate`
生成网络，执行下列函数
```bash
	generateCerts
	replacePrivateKey
	generateChannelArtifacts
```
### 2.1 `generateCerts`
先判断`cryptogen`二进制文件是否存在，存在则执行
```bash
cryptogen generate --config=./crypto-config.yaml
```
生成证书，接下来我们看看`crypto-config.yaml`文件。crypto-config.yaml 文件中定义了两个 Org —— Peer 与Orderer，里面的具体内容还没看懂。
我们来运行一下 cryptogen 程序，运行之前可以看到 `first-network`文件夹下的文件情况如下：
```
	 first-network
		|- /base
		|- /channel-artifacts
		|- /scripts
		|- bysh.sh
		|- configtx.yaml
		|- crypto-config.yaml
		|- docker-compose-cli.uaml
		|- docker-compose-couch.yaml
		|- docker-compose-e2e-template.yaml
		|- README.MD
```
运行`cryptogen`之后，`first-network`文件夹下新出现了`crypto-config`文件夹，其结构如下：
```
	 crypto-config
		|- /orderOrganizations
		|	|- /example.com
		|		|- /ca
		|		|- /msp
		|		|- /orderers
		|		|- /tlsca
		|		|- /users
		|
		|- /peerOrganizations
		|	|- /org1.example.com
		|		|- /ca
		|		|- /msp
		|		|- /peers
		|		|- /tlsca
		|		|- /users
		|
		|	|- /org2.example.com
		
```
相关文件夹下的内容省略。可以看到`crypto-config.yaml`文件中的`Domain`字段，就对应各个Org下的文件夹名称，例如，对`PeerOrgs`中的`Org1`，其文件夹名称就是`org1.example.com`

我们来看一下各`ca`文件夹下的`*.pem`文件的内容，执行`openssl x509 -in <key_name>.pem -noout -text`。与证书文件同目录下有一个`<name>_sk`文件，这个文件是私钥文件
### 2.2 `replacePrivateKey`
先检查系统的内核名称，因为在MacOSX上的sed不支持带有null扩展名的-i标志。将`docker-compose-e2e-template.yaml`复制为`docker-compose-e2e.yaml`，将`docker-compose-e2e.yaml`中的`CA1_PRIVATE_KEY`与`CA2_PRIVATE_KEY`替换为`org1`与`org2`中`ca`文件夹下的对应的私钥文件的名称。
？`docker-compose-e2e.yaml`文件后面就没有用到了，这部分的操作目的是什么呢？还是说我没有找到使用`docker-compose-e2e.yaml`的地方？？？？
### 2.3 `generateChannelArtifacts`
`generateChannelArtifacts`主要就是执行了下面四条操作：
- `configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block`
- `configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME`
- `configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP`
- `configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate   ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP`

四条操作分别意味着：
- 生成创世区块
- 生成 channeltx
- 更新Org1MSP的锚节点
- 更新Org2MSP的锚节点

上述四个操作执行完后，`/first-network/channel-artifacts`下新建了下列四个文件：
- genesis.block
- channel.tx
- Org1MSPanchors.tx
- Org2MSPanchors.tx

## 3. `./byfn -m up`
执行`networkUp`函数，执行`CHANNEL_NAME=$CHANNEL_NAME TIMEOUT=$CLI_TIMEOUT docker-compose -f $COMPOSE_FILE up -d 2>&1`开启各个容器，启动网络































