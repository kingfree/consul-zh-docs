# 运行Consul Agent

在安装完Consul之后，我们需要运行Consul agent。在本教程中，你将在开发模式下运行它，开发模式是一种不安全以及不可扩展的模式，但是可以让你不需要额外的配置就能够尝试Consul的大部分功能。你还将学习到如何优雅的关闭Consul agent。

#### Server agent 和 Client agent

在生产环境下，你应当以Server或Client模式运行每个Consul agent。每个Consul数据中心必须至少有一个Server，该Server负责维护Consul的状态，这包括有关其它Consul server和Consul client的信息，可用于发现的服务以及允许哪些服务与哪些其他服务进行通信的信息。

> 警告：我们强烈不建议使用单服务器在生产环境部署。


为了确保即使某个Consul Server agent发生故障，Consul的状态也会保留，你应该始终在生产环境中运行3~5台Server agent。 使用不超过5个并且奇数数量的Server agent可以在性能和容错能力之间取得平衡。 你可以在Consul的体系结构文档中了解有关这些要求的更多信息。


除了以Server模式运行的Consul agent外，还有以Client模式运行的。 Client agent是一个轻量级的进程，用于注册服务，运行状况检查并将服务查询转发到Server agent。 Client agent必须在Consul数据中心中运行服务的每个节点上运行，因为Client agent是有关服务运行状况的真实来源。

当你准备在生产环境中使用Consul的时候，你可以在我们的开发指南中找到有关Server agent和Client agent在生产环境部署的更多指南。现在，为了便于学习，让我们以开发模式在本地启动agent。

> 警告：永远不要在生产环境中运行  -dev 指令。


#### 启动Consul agent
以开发模式启动Consul agent

	$ consul agent -dev


	==> Starting Consul agent...
	           Version: '1.8.1'
	           Node ID: '7a82fb12-a621-004b-a8cd-98861b80d64c'
	         Node name: 'C02YM00WJG5H.local' //Consul默认使用主机名作为Nodename
	        Datacenter: 'dc1' (Segment: '<all>')
	            Server: true (Bootstrap: false)
	       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
	      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
	           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false, Auto-Encrypt-TLS: false
	
	==> Log data will now stream in as it occurs:
	
	    2020-08-07T13:43:09.554+0800 [DEBUG] agent: Using random ID as node ID: id=7a82fb12-a621-004b-a8cd-98861b80d64c
	    2020-08-07T13:43:09.561+0800 [WARN]  agent: Node name will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.: node_name=C02YM00WJG5H.local
	    2020-08-07T13:43:09.568+0800 [INFO]  agent.server.raft: initial configuration: index=1 servers="[{Suffrage:Voter ID:7a82fb12-a621-004b-a8cd-98861b80d64c Address:127.0.0.1:8300}]"
	    2020-08-07T13:43:09.569+0800 [INFO]  agent.server.raft: entering follower state: follower="Node at 127.0.0.1:8300 [Follower]" leader=
	    2020-08-07T13:43:09.570+0800 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: C02YM00WJG5H.local.dc1 127.0.0.1
	    2020-08-07T13:43:09.571+0800 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: C02YM00WJG5H.local 127.0.0.1
	    2020-08-07T13:43:09.572+0800 [INFO]  agent.server: Handled event for server in area: event=member-join server=C02YM00WJG5H.local.dc1 area=wan
	    2020-08-07T13:43:09.572+0800 [INFO]  agent.server: Adding LAN server: server="C02YM00WJG5H.local (Addr: tcp/127.0.0.1:8300) (DC: dc1)"
	    2020-08-07T13:43:09.575+0800 [INFO]  agent: Started DNS server: address=127.0.0.1:8600 network=tcp
	    2020-08-07T13:43:09.575+0800 [INFO]  agent: Started DNS server: address=127.0.0.1:8600 network=udp
	    2020-08-07T13:43:09.575+0800 [INFO]  agent: Started HTTP server: address=127.0.0.1:8500 network=tcp
	    2020-08-07T13:43:09.576+0800 [INFO]  agent: Started gRPC server: address=127.0.0.1:8502 network=tcp
	    2020-08-07T13:43:09.576+0800 [INFO]  agent: started state syncer
	==> Consul agent running!
	    2020-08-07T13:43:09.635+0800 [WARN]  agent.server.raft: heartbeat timeout reached, starting election: last-leader=
	    2020-08-07T13:43:09.636+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 127.0.0.1:8300 [Candidate]" term=2
	    2020-08-07T13:43:09.636+0800 [DEBUG] agent.server.raft: votes: needed=1
	    2020-08-07T13:43:09.636+0800 [DEBUG] agent.server.raft: vote granted: from=7a82fb12-a621-004b-a8cd-98861b80d64c term=2 tally=1
	    2020-08-07T13:43:09.636+0800 [INFO]  agent.server.raft: election won: tally=1
	    2020-08-07T13:43:09.636+0800 [INFO]  agent.server.raft: entering leader state: leader="Node at 127.0.0.1:8300 [Leader]"
	    2020-08-07T13:43:09.636+0800 [INFO]  agent.server: cluster leadership acquired
	    2020-08-07T13:43:09.637+0800 [INFO]  agent.server: New leader elected: payload=C02YM00WJG5H.local
	    2020-08-07T13:43:09.637+0800 [DEBUG] agent.server: Cannot upgrade to new ACLs: leaderMode=0 mode=0 found=true leader=127.0.0.1:8300
	    2020-08-07T13:43:09.642+0800 [DEBUG] connect.ca.consul: consul CA provider configured: id=07:80:c8:de:f6:41:86:29:8f:9c:b8:17:d6:48:c2:d5:c5:5c:7f:0c:03:f7:cf:97:5a:a7:c1:68:aa:23:ae:81 is_primary=true
	    2020-08-07T13:43:09.657+0800 [INFO]  agent.server.connect: initialized primary datacenter CA with provider: provider=consul
	    2020-08-07T13:43:09.657+0800 [INFO]  agent.leader: started routine: routine="federation state anti-entropy"
	    2020-08-07T13:43:09.657+0800 [INFO]  agent.leader: started routine: routine="federation state pruning"
	    2020-08-07T13:43:09.657+0800 [INFO]  agent.leader: started routine: routine="CA root pruning"
	    2020-08-07T13:43:09.657+0800 [DEBUG] agent.server: Skipping self join check for node since the cluster is too small: node=C02YM00WJG5H.local
	    2020-08-07T13:43:09.657+0800 [INFO]  agent.server: member joined, marking health alive: member=C02YM00WJG5H.local
	    2020-08-07T13:43:09.658+0800 [INFO]  agent.server: federation state anti-entropy synced
	    2020-08-07T13:43:09.716+0800 [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
	    2020-08-07T13:43:09.716+0800 [INFO]  agent: Synced node info
	    2020-08-07T13:43:09.797+0800 [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
	    2020-08-07T13:43:09.797+0800 [DEBUG] agent: Node info in sync
	    2020-08-07T13:43:09.797+0800 [DEBUG] agent: Node info in sync
	    2020-08-07T13:44:09.638+0800 [DEBUG] agent.server: Skipping self join check for node since the cluster is too small: node=C02YM00WJG5H.local
	    2020-08-07T13:44:12.669+0800 [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
	    2020-08-07T13:44:12.669+0800 [DEBUG] agent: Node info in sync
	    2020-08-07T13:45:09.575+0800 [DEBUG] agent.server.router.manager: No healthy servers during rebalance, aborting
	    2020-08-07T13:45:09.639+0800 [DEBUG] agent.server: Skipping self join check for node since the cluster is too small: node=C02YM00WJG5H.local
	    2020-08-07T13:45:45.883+0800 [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
	    2020-08-07T13:45:45.883+0800 [DEBUG] agent: Node info in sync
	    2020-08-07T13:46:09.640+0800 [DEBUG] agent.server: Skipping self join check for node since the cluster is too small: node=C02YM00WJG5H.local

日志显式Consul agent已启动并且正在传输一些日志流数据。日志还显式该Consul agent正在以Server模式运行，并且被选举为Leader。此外，本地agent已被标记为数据中心的正常成员。从上面可以看出我们在后续查询服务时需要用到的端口信息：

    Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
    Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)

> OS X用户注意事项：Consul使用你的主机名作为默认节点名。 如果你的主机名包含句点，则对该节点的DNS查询将不适用于Consul。 为避免这种情况，请使用`-node`标志显式设置节点的名称。

#### 查找数据中心成员
通过在新的终端窗口中运行`consul menbers`命令，可以检查Consul数据中心的成员。 该命令的输出结果列出了数据中心中所有的agent。 稍后我们将介绍将其它Consul agent一起加入到数据中心的方法，但是目前数据中心中只有一个成员（你的计算机）。

	$ consul members
	
	Node                Address         Status  Type    Build  Protocol  DC  Segment
	C02YM00WJG5H.local  127.0.0.1:8301  alive   server  1.8.1  2         dc1  <all>

该输出展示了你在上一个窗口启动的Consul agent, 它的地址，健康状态，它在数据中心的状态以及一些版本信息。你也可以通过`-detailed`标志来展示更多元信息。

同时你也可以在另一个窗口看到一次查询日志：

	2020-08-07T13:56:57.969+0800 [DEBUG] agent.http: Request finished: method=GET url=/v1/agent/members?segment=_all from=127.0.0.1:55189 latency=265.761µs

`menbers`指令是针对Consul client运行的，通过流言协议获取client的信息。client拥有的信息最终是一致的，但是在任何时间点，其对全局状态的视角都可能与Server agent上的状态不完全匹配。 要获得与全局高度一致的视图，请调用HTTP API，该请求会将请求转发到Consul Server agent。

	$ curl localhost:8500/v1/catalog/nodes

#
	
	[
	    {
	        "ID": "7a82fb12-a621-004b-a8cd-98861b80d64c",
	        "Node": "C02YM00WJG5H.local",
	        "Address": "127.0.0.1",
	        "Datacenter": "dc1",
	        "TaggedAddresses": {
	            "lan": "127.0.0.1",
	            "lan_ipv4": "127.0.0.1",
	            "wan": "127.0.0.1",
	            "wan_ipv4": "127.0.0.1"
	        },
	        "Meta": {
	            "consul-network-segment": ""
	        },
	        "CreateIndex": 10,
	        "ModifyIndex": 12
	    }
	]

除了HTTP API之外，你还可以使用DNS接口来查找节点。 除非你启用了缓存，否则DNS接口会将你的查询请求发送到Consul Server。 要执行DNS查找，你必须指向Consul agent的DNS服务器，该服务器默认在**8600**端口上运行。 DNS参数的格式（例如Judiths-MBP.node.consul）将在后面详细介绍。

	$ dig @127.0.0.1 -p 8600 Judiths-MBP.node.consul
	
	; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 Judiths-MBP.node.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7104
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
	;; WARNING: recursion requested but not available
	
	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;Judiths-MBP.node.consul.   IN  A
	
	;; ANSWER SECTION:
	Judiths-MBP.node.consul. 0  IN  A   127.0.0.1
	
	;; ADDITIONAL SECTION:
	Judiths-MBP.node.consul. 0  IN  TXT "consul-network-segment="
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Mon Jul 15 19:43:58 PDT 2019
	;; MSG SIZE  rcvd: 104

#### 停止Consol agent
可以使用`consul leave`命令停止consul agent。该指令会优雅的停止Consul agent，使其离开Consul数据中心并关闭。

	$ consul leave
	Graceful leave complete

如果你切换回之前的那个带有Consul日志流输出的窗口，可以看到这些日志显示Consul agent已离开数据中心。

	2020-08-07T14:15:19.445+0800 [INFO]  agent.server: server starting leave
	    2020-08-07T14:15:19.445+0800 [INFO]  agent.server.serf.wan: serf: EventMemberLeave: C02YM00WJG5H.local.dc1 127.0.0.1
	    2020-08-07T14:15:19.445+0800 [INFO]  agent.server: Handled event for server in area: event=member-leave server=C02YM00WJG5H.local.dc1 area=wan
	    2020-08-07T14:15:19.445+0800 [INFO]  agent.server.router.manager: shutting down
	    2020-08-07T14:15:22.446+0800 [INFO]  agent.server.serf.lan: serf: EventMemberLeave: C02YM00WJG5H.local 127.0.0.1
	    2020-08-07T14:15:22.446+0800 [INFO]  agent.server: Removing LAN server: server="C02YM00WJG5H.local (Addr: tcp/127.0.0.1:8300) (DC: dc1)"
	    2020-08-07T14:15:22.446+0800 [WARN]  agent.server: deregistering self should be done by follower: name=C02YM00WJG5H.local
	    2020-08-07T14:15:23.665+0800 [ERROR] agent.server.autopilot: Error updating cluster health: error="error getting server raft protocol versions: No servers found"
	    2020-08-07T14:15:25.447+0800 [INFO]  agent.server: Waiting to drain RPC traffic: drain_time=5s
	    2020-08-07T14:15:25.666+0800 [ERROR] agent.server.autopilot: Error updating cluster health: error="error getting server raft protocol versions: No servers found"
	    2020-08-07T14:15:27.666+0800 [ERROR] agent.server.autopilot: Error updating cluster health: error="error getting server raft protocol versions: No servers found"
	    2020-08-07T14:15:29.666+0800 [ERROR] agent.server.autopilot: Error updating cluster health: error="error getting server raft protocol versions: No servers found"
	    2020-08-07T14:15:29.666+0800 [ERROR] agent.server.autopilot: Error promoting servers: error="error getting server raft protocol versions: No servers found"
	    2020-08-07T14:15:30.447+0800 [INFO]  agent: Requesting shutdown
	    2020-08-07T14:15:30.447+0800 [INFO]  agent.server: shutting down server
	    2020-08-07T14:15:30.447+0800 [DEBUG] agent.leader: stopping routine: routine="federation state anti-entropy"
	    2020-08-07T14:15:30.447+0800 [DEBUG] agent.leader: stopping routine: routine="federation state pruning"
	    2020-08-07T14:15:30.447+0800 [DEBUG] agent.leader: stopping routine: routine="CA root pruning"
	    2020-08-07T14:15:30.447+0800 [ERROR] agent.server: error performing anti-entropy sync of federation state: error="context canceled"
	    2020-08-07T14:15:30.447+0800 [DEBUG] agent.leader: stopped routine: routine="federation state pruning"
	    2020-08-07T14:15:30.447+0800 [DEBUG] agent.leader: stopped routine: routine="CA root pruning"
	    2020-08-07T14:15:30.447+0800 [DEBUG] agent.leader: stopped routine: routine="federation state anti-entropy"
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: consul server down
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: shutdown complete
	    2020-08-07T14:15:30.448+0800 [DEBUG] agent.http: Request finished: method=PUT url=/v1/agent/leave from=127.0.0.1:59821 latency=11.002045748s
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: Stopping server: protocol=DNS address=127.0.0.1:8600 network=tcp
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: Stopping server: protocol=DNS address=127.0.0.1:8600 network=udp
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: Stopping server: protocol=HTTP address=127.0.0.1:8500 network=tcp
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: Waiting for endpoints to shut down
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: Endpoints down
	    2020-08-07T14:15:30.448+0800 [INFO]  agent: Exit code: code=0

当发出`leave`命令的时候，Consul会通知其他成员该agent离开了数据中心。 agent离开后，在同一节点上运行的本地服务以及对其的检查将从目录中删除，并且Consul不会尝试再次与该节点联系。

如果强制终止agent进程则会向Consul数据中心中的其他agent指示该节点发生故障而不是正常离开。 当节点发生故障时，其运行状况将标记为“critical”，但不会从目录中删除。 Consul将自动尝试重新连接到发生故障的节点，前提是该节点由于网络原因可能暂时不可用，并且可能会恢复回来。

如果agent程序正在以Server模式运行，那么优雅离开就变得很重要，这样可以避免导致潜在的可用性中断进而而影响一致性（https://www.consul.io/docs/internals/consensus.html）。 查看“添加和删除服务（https://learn.hashicorp.com/consul/day-2-operations/servers）”教程，以获取有关如何安全添加和删除Server的详细信息。

#### 下一步

现在，你已经在开发模式下学习了启动和停止了Consul agent，请继续下一个教程，通过Consul 服务发现注册服务，你将在其中学习Consul是如何知道某服务在数据中心中是否存在及其所在位置的。
