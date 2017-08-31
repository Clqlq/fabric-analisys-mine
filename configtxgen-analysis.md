## configtxgen 源码分析
### 1. 文件结构

	configtxgen
	|-	localconfig
	|		config.go
	|-	metadata
	|		metadata.go
	|		metadata_test.go
	|-	provisional
	|		provisional.go
	|		provisional_test.go
	|-	main.go
	|-	main_test.go

### 2. 代码分析
#### 2.1 命令行参数解析
首先用**flag**包 的 **StringVar** 与 **Parse** 方法进行命令行参数解析，分别设置了下面几个参数

	- outputBlock
	- channelID
	- outputChannelCreateTx
	- profile
	- inspectBlock
	- inspectChannelCreateTx
	- outputAnchorPeersUpdate
	- asOrg
	- version(默认为false)

若version设置为true，则调用`func printVersion()`函数打印相关版本信息，然后执行`os.Exit(exitCode)`退出`main`函数。其中，`func printVersion()`函数定义在`metadata.go`文件中，`metadata.go`文件中只定义了这一个函数。

接着使用`defer`关键字，调用一个匿名函数，如果有`error`则打印一些错误信息

#### 2.2 函数主要部分
##### 2.2.1 
- 调用`bccsp/factory/pkcs11.go`下的`InitFactories()`，该函数的输入参数类型为`FactotyOpts`结构体的指针，返回值为一个`error`类型，`FactotyOpts`结构体定义在`pkcs11.go`文件中。我们传给`InitFactories()`的参数是`nil`，但是没有接收其返回值。这部分的操作所产生的结构体的指针丢失了
##### 2.2.2	
- 执行`config := genesisconfig.Load(profile)`

	- 其中`profile`由命令行参数解析而得，默认值为`genesisconfig.SampleInsecureProfile`，`genesisconfig`就是`/configtxgen/localconfig`路径，该路径下只有`config.go`文件，`SampleInsecureProfile`的定义为`const SampleInsecureProfile = "SampleInsecureSolo"`
	
	- `SampleInsecureSolo`定义了一个不含MSP定义的`Solo orderer`，并且允许所有的`transaction`与`channel`向联盟`SampleConsortium`创建请求。其结构如下代码段所示：
	


			    SampleInsecureSolo:
					Orderer:
				    		<<: *OrdererDefaults
					Consortiums:
				    		SampleConsortium:
							Organizations:

	        
	- `Load()`函数输入参数为字符串，输出参数为`Profile`结构体的指针。`Load`返回的是给定配置文件的`orderer`/`application`的配置组合，`Load`做的主要操作都是一些路径的搜寻	
##### 2.2.3
然后`main`函数执行下面的代码块：

		if outputBlock != "" {
			if err := doOutputBlock(config, channelID, outputBlock); err != nil {
				logger.Fatalf("Error on outputBlock: %s", err)
			}
		}

		if outputChannelCreateTx != "" {
			if err := doOutputChannelCreateTx(config, channelID, outputChannelCreateTx); err != nil {
				logger.Fatalf("Error on outputChannelCreateTx: %s", err)
			}
		}

		if inspectBlock != "" {
			if err := doInspectBlock(inspectBlock); err != nil {
				logger.Fatalf("Error on inspectBlock: %s", err)
			}
		}

		if inspectChannelCreateTx != "" {
			if err := doInspectChannelCreateTx(inspectChannelCreateTx); err != nil {
				logger.Fatalf("Error on inspectChannelCreateTx: %s", err)
			}
		}

		if outputAnchorPeersUpdate != "" {
			if err := doOutputAnchorPeersUpdate(config, channelID, outputAnchorPeersUpdate, asOrg); err != nil {
				logger.Fatalf("Error on inspectChannelCreateTx: %s", err)
			}
		}


- `doOutputBlick()`函数：用来生成创世区块
	
	- 执行`pgen := provisional.New(config)`，`provisional`是`/commom/tools/configtxgen/provisional`包，下面只有一个文件`provisional.go`，其中`New()`函数的返回类型是`Generator interface`，实际返回了一个`bootstrapper`结构体的指针，函数说明说`New returns a new provisional bootstrap helper.`但具体实现没看懂
	
	- 相关的错误信息检测，然后执行`genesisBlock := pgen.GenesisBlockForChannel(channelID)`，得到了对应通道ID的创世区块。
	
	- 执行`err := ioutil.WriteFile(outputBlock, utils.MarshalOrPanic(genesisBlock), 0644)`，将得到的区块写入之前给定的路径
	
	- 相关错误信息检测，返回

- `doOutputChannelCreateTx`函数：生成通道的配置交易（config transaction)： 区块链网络的共享配置存储在每个channel的配置交易集合中，配置交易简称 configtx
	
	- 主要是执行了`configtx, err := channelconfig.MakeChainCreationTransaction(channelID, conf.Consortium, nil, orgNames...)`，`MakeChainCreationTransaction`函数定义在`commom/config/channel/template.go`中，函数的主要作用就是通过使用模块框架 新建了一个通道事物（channel transaction），返回的是定义在`protos/common/configtx.pb.go`里的`ConfigUpdateEnvelope`结构体的指针
	
	- 执行`err = ioutil.WriteFile(outputChannelCreateTx, utils.MarshalOrPanic(configtx), 0644)`，把交易给写下来
	
	-后面就是一些错误检测，返回`nil`

- `doOutputAnchorPeersUpdate`函数：执行了锚节点更新
	
	- 找到对应的`Organization`
	
	- 执行`anchorPeers := make([]*pb.AnchorPeer, len(org.AnchorPeers))`，新建`anchorPeers`节点，存储相应的信息
	
	- `AnchorPeersUpdate`中包含的信息如下：
	
				- {
					- Payload
						- Header
							- ChannelHeader
							- ChannelId
							- Type
						- Data
		
					}
	
	- `err := ioutil.WriteFile(outputAnchorPeersUpdate, utils.MarshalOrPanic(update), 0644)`写锚节点
	
	
	
- `doInspectBlock`函数
	- `logger.Info("Inspecting block")`
	
	- 区块的具体结构如下所示：
		- `BlockHeader`：BlockHeader是形成块链的块的元素。 块头使用配置的散列算法进行散列，通过BlockHeader的ASN.1编码
		- `BlockData`：byte 类型的二位数组
		- `BlockMetaData`：byte 类型的二位数组
	

				type Block struct {
					Header   	*BlockHeader   `protobuf:"bytes,1,opt,name=header" json:"header,omitempty"`
					Data    	 *BlockData     `protobuf:"bytes,2,opt,name=data" json:"data,omitempty"`
					Metadata *BlockMetadata `protobuf:"bytes,3,opt,name=metadata" json:"metadata,omitempty"`
				}
				
	- 区块示例输出如下：
	
			Config for channel: foo
			{
			    "": {
				"Values": {},
				"Groups": {
				    "/Channel": {
					"Values": {
					    "HashingAlgorithm": {
						"Version": "0",
						"ModPolicy": "",
						"Value": {
						    "name": "SHA256"
						}
					    },
					    "BlockDataHashingStructure": {
						"Version": "0",
						"ModPolicy": "",
						"Value": {
						    "width": 4294967295
						}
					    },
					    "OrdererAddresses": {
						"Version": "0",
						"ModPolicy": "",
						"Value": {
						    "addresses": [
						        "127.0.0.1:7050"
						    ]
						}
					    }
					},
					"Groups": {
					    "/Channel/Orderer": {
						"Values": {
						    "ChainCreationPolicyNames": {
						        "Version": "0",
						        "ModPolicy": "",
						        "Value": {
						            "names": [
						                "AcceptAllPolicy"
						            ]
						        }
						    },
						    "ConsensusType": {
						        "Version": "0",
						        "ModPolicy": "",
						        "Value": {
						            "type": "solo"
						        }
						    },
						    "BatchSize": {
						        "Version": "0",
						        "ModPolicy": "",
						        "Value": {
						            "maxMessageCount": 10,
						            "absoluteMaxBytes": 103809024,
						            "preferredMaxBytes": 524288
						        }
						    },
						    "BatchTimeout": {
						        "Version": "0",
						        "ModPolicy": "",
						        "Value": {
						            "timeout": "10s"
						        }
						    },
						    "IngressPolicyNames": {
						        "Version": "0",
						        "ModPolicy": "",
						        "Value": {
						            "names": [
						                "AcceptAllPolicy"
						            ]
						        }
						    },
						    "EgressPolicyNames": {
						        "Version": "0",
						        "ModPolicy": "",
						        "Value": {
						            "names": [
						                "AcceptAllPolicy"
						            ]
						        }
						    }
						},
						"Groups": {}
					    },
					    "/Channel/Application": {
						"Values": {},
						"Groups": {}
					    }
					}
				    }
				}
			    }
			}
	
	
	
- `doInspectChannelCreateTx`函数
	- `logger.Info("Inspecting transaction")`

	- `channeltx`内包含的信息的结构如如下：
	
			c_lq@ubuntu:~/hyex/fabric-samples/first-network$ ../bin/configtxgen -profile TwoOrgsOrdererGenesis -inspectChannelCreateTx ./channel-artifacts/channel.tx 
			2017-08-30 15:01:00.102 CST [common/configtx/tool] main -> INFO 001 Loading configuration
			2017-08-30 15:01:00.110 CST [common/configtx/tool] doInspectChannelCreateTx -> INFO 002 Inspecting transaction
			2017-08-30 15:01:00.110 CST [common/configtx/tool] doInspectChannelCreateTx -> INFO 003 Parsing transaction

			Channel creation for channel: mychannel

			Read Set:
			{
			    "Channel": {
				"Values": {
				    "Consortium": {
					"Version": "0",
					"ModPolicy": "",
					"Value": {
					    "name": "SampleConsortium"
					}
				    }
				},
				"Policies": {},
				"Groups": {
				    "Application": {
					"Values": {},
					"Policies": {},
					"Groups": {
					    "Org2MSP": {
						"Values": {},
						"Policies": {},
						"Groups": {}
					    },
					    "Org1MSP": {
						"Values": {},
						"Policies": {},
						"Groups": {}
					    }
					}
				    }
				}
			    }
			}

			Write Set:
			{
			    "Channel": {
				"Values": {
				    "Consortium": {
					"Version": "0",
					"ModPolicy": "",
					"Value": {
					    "name": "SampleConsortium"
					}
				    }
				},
				"Policies": {},
				"Groups": {
				    "Application": {
					"Values": {},
					"Policies": {
					    "Readers": {
						"Version": "0",
						"ModPolicy": "Admins",
						"Policy": {
						    "PolicyType": "3",
						    "Policy": {
						        "subPolicy": "Readers",
						        "rule": "ANY"
						    }
						}
					    },
					    "Admins": {
						"Version": "0",
						"ModPolicy": "Admins",
						"Policy": {
						    "PolicyType": "3",
						    "Policy": {
						        "subPolicy": "Admins",
						        "rule": "MAJORITY"
						    }
						}
					    },
					    "Writers": {
						"Version": "0",
						"ModPolicy": "Admins",
						"Policy": {
						    "PolicyType": "3",
						    "Policy": {
						        "subPolicy": "Writers",
						        "rule": "ANY"
						    }
						}
					    }
					},
					"Groups": {
					    "Org1MSP": {
						"Values": {},
						"Policies": {},
						"Groups": {}
					    },
					    "Org2MSP": {
						"Values": {},
						"Policies": {},
						"Groups": {}
					    }
					}
				    }
				}
			    }
			}

			Delta Set:
			[Policy] /Channel/Application/Writers
			[Policy] /Channel/Application/Readers
			[Policy] /Channel/Application/Admins
			[Groups] /Channel/Application

	
	
### 3. 总结
`configtxgen`部分代码主要完成了三个工作，一个是生成对应通道的创世区块，一个是生成了通道的配置交易（configtx），最后一个就是更新了通道的锚节点












