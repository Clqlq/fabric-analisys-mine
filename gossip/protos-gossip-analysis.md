先看一下 /protos/gossip/ 下的文件

message.proto 里面基本定义了在 gossip 中所需要用到的消息类型，包括了各种类型。


```proto

	// Gossip 定义了一个 Gossip 服务，其中包含了两个 rpc 服务
	service Gossip {

	    // GossipStream 是 gRPC stream 用来发送与接收信息
	    rpc GossipStream (stream Envelope) returns (stream Envelope) {}

	    // Ping 是用来探测远程 peer 的存活情况
	    rpc Ping (Empty) returns (Empty) {}
	}

	// Envelope 包括了一个序列化了的 GossipMessage 信息
	// 以及对该信息的一个签名
	// 它也可能包括一个 SecretEnvelope —— 一个序列化的 Secret
	message Envelope {
	    bytes payload   = 1;
	    bytes signature = 2;
	    SecretEnvelope secret_envelope = 3;
	}


	// Secret 是一个序列化的 Secret 以及对该 Secret 的一个签名
	// 签名应该由签署 SecretEnvelope 的同行验证
	message SecretEnvelope {
	    bytes payload   = 1;
	    bytes signature = 2;
	}

	// Secret 是一个可以从 Envelope 中被忽略的实体
	// 当远程 peer 接收了 Envelope 且不应该知道 secret 的内容时
	// peer 就将 Secret 忽略
	message Secret {
	    oneof content {
		string internalEndpoint = 1;
	    }
	}

	// GossipMessage 定义了在 gossip 网络中传送的消息
	message GossipMessage {

	    // 主要用于测试，但将来可能会用于确保消息传递
	    uint64 nonce  = 1;

	    // 消息的通道
	    // 一些 GossipMessages 可能会将其置为 nil 因为它们是跨通道的
	    // 但另一些也可能不会
	    bytes channel = 2;


	    enum Tag {
			UNDEFINED    = 0;
			EMPTY        = 1;
			ORG_ONLY     = 2;
			CHAN_ONLY    = 3;
			CHAN_AND_ORG = 4;
			CHAN_OR_ORG  = 5;
	    }

	    // 决定哪些 peers 可以转发消息
	    Tag tag = 3;

	    oneof content {
			// Membership 成员
			AliveMessage alive_msg = 5;
			MembershipRequest mem_req = 6;
			MembershipResponse mem_res = 7;

			// Contains a ledger block 包含一个账本区块
			DataMessage data_msg = 8;

			// 用来 push&pull
			GossipHello hello = 9;
			DataDigest  data_dig = 10;
			DataRequest data_req = 11;
			DataUpdate  data_update = 12;

			// 空信息，用来 ping
			Empty empty = 13;

			// ConnEstablish 用来建立连接
			ConnEstablish conn = 14;

			// 用来中继关于 state 的信息
			StateInfo state_info = 15;

			// 用来发送 StateInfo messages 集合
			StateInfoSnapshot state_snapshot = 16;

			//用来询问 StateInfoSnapshots
			StateInfoPullRequest state_info_pull_req = 17;

			//  用来向远程 peer 请求 blocks 集合
			RemoteStateRequest state_request = 18;

			// 用来向远程 peer 发送 blocks 集合
			RemoteStateResponse state_response = 19;

			// 用来表明 peer 成为 leader 的意图
			LeadershipMessage leadership_msg = 20;

			// 用来学习一个 peer 的 certificate
			PeerIdentity peer_identity = 21;

			Acknowledgement ack = 22;

			// 用来请求私有数据
			RemotePvtDataRequest privateReq = 23;

			// 用来回复私有数据请求
			RemotePvtDataResponse privateRes = 24;

			// 打包将要分发的私有数据
			// 在背书后的私读写集
			PrivateDataMessage private_data = 25;
			}
	}

```

剩下的就基本是上面 GossipMessage 内部各种信息类型的具体结构，不赘述了。










