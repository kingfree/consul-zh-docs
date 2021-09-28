# 2. 运行 Consul Agent

安装完 Consul 后应该运行 agent，它可以运行在服务端\(server\)或者客户端\(client\)模式下。每个数据中心\(datacenter\)都至少有一个 server，推荐一个集群\(cluster\)至少有 3 到 5 个 server。因为在故障情况下数据丢失是不可避免的，所以**强烈**建议不要单机部署。

其他 agent 运行在 client 模式，它是一个轻量级进程，提供服务注册、健康检查和服务器间的查询转发。集群的所有节点都应运行一个 agent。

关于如何启动数据中心，请见[进阶教程（一）](https://kingfree.gitbook.io/consul/jin-jie-jiao-cheng-yi-bu-shu-shu-ju-zhong-xin)。

### 启动 Agent

简单起见，我们用开发模式启动 Consul agent，该模式可以快速便捷地创建好单节点 Consul 环境。请**不要**将开发模式用在生产环境上，因为它不会保存任何状态。

```text
$ consul agent -dev
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
           Version: 'v0.7.0'
         Node name: 'Armons-MacBook-Air'
        Datacenter: 'dc1'
            Server: true (bootstrap: false)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2016/09/15 10:21:10 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:127.0.0.1:8300 Address:127.0.0.1:8300}]
    2016/09/15 10:21:10 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2016/09/15 10:21:10 [INFO] serf: EventMemberJoin: Armons-MacBook-Air 127.0.0.1
    2016/09/15 10:21:10 [INFO] serf: EventMemberJoin: Armons-MacBook-Air.dc1 127.0.0.1
    2016/09/15 10:21:10 [INFO] consul: Adding LAN server Armons-MacBook-Air (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2016/09/15 10:21:10 [INFO] consul: Adding WAN server Armons-MacBook-Air.dc1 (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2016/09/15 10:21:13 [DEBUG] http: Request GET /v1/agent/services (180.708Âµs) from=127.0.0.1:52369
    2016/09/15 10:21:13 [DEBUG] http: Request GET /v1/agent/services (15.548Âµs) from=127.0.0.1:52369
    2016/09/15 10:21:17 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2016/09/15 10:21:17 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2016/09/15 10:21:17 [DEBUG] raft: Votes needed: 1
    2016/09/15 10:21:17 [DEBUG] raft: Vote granted from 127.0.0.1:8300 in term 2. Tally: 1
    2016/09/15 10:21:17 [INFO] raft: Election won. Tally: 1
    2016/09/15 10:21:17 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2016/09/15 10:21:17 [INFO] consul: cluster leadership acquired
    2016/09/15 10:21:17 [DEBUG] consul: reset tombstone GC to index 3
    2016/09/15 10:21:17 [INFO] consul: New leader elected: Armons-MacBook-Air
    2016/09/15 10:21:17 [INFO] consul: member 'Armons-MacBook-Air' joined, marking health alive
    2016/09/15 10:21:17 [INFO] agent: Synced service 'consul'
```

如你所见，Consul agent 启动输出了一些日志数据。你可以看到 agent 运行在 server 模式下，并成为该集群的 leader。另外，本地成员被标记为了该集群的健康成员。

### 集群成员

你可以通过 `consul members` 命令查看 Consul 集群的成员。现在你只能看到一个成员，下一节会介绍如何加入集群：

```text
$ consul members
Node                Address         Status  Type    Build  Protocol  DC   Segment
Armons-MacBook-Air  127.0.0.1:8301  alive   server  1.4.0  2         dc1  <all>
```

输出的是我们自己的节点，包括它的运行地址、健康状态、在集群的角色和版本信息。更多数据可以用  `-detailed` 参数获取。

命令 `members` 的输出基于 [gossip 协议](https://consul.io/docs/internals/gossip.html)，它们是“结果一致”。这意味着，在此时，你从屏幕上看到的节点状态不一定是 server 节点的真实状态。如果想要更健壮的一致性视图，请使用 [HTTP API](https://consul.io/api/index.html) 来请求 Consul 服务器：

```text
$ curl localhost:8500/v1/catalog/nodes
[
    {
        "ID": "bac74f08-b3c0-e37e-c644-92f394659196",
        "Node": "Armons-MacBook-Air",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "Meta": {
            "consul-network-segment": ""
        },
        "CreateIndex": 7,
        "ModifyIndex": 7
    }
]
```

作为 HTTP API 的补充，[DNS 接口](https://consul.io/docs/agent/dns.html)可以用于查询节点。注意，你需要保证你的 DNS 查询通过 Consul agent 默认运行在 8600 端口的 DNS 服务。稍后将详细介绍这些 DNS 项的含义。

```text
$ dig @127.0.0.1 -p 8600 Armons-MacBook-Air.node.consul
;; QUESTION SECTION:
;Armons-MacBook-Air.node.consul.    IN  A

;; ANSWER SECTION:
Armons-MacBook-Air.node.consul. 0 IN    A   127.0.0.1
```

### 停止 Agent

你可以直接用 `Ctrl-C` 来平滑退出，你将会看到它离开集群并关闭。

通过平滑退出，Consul 会通知其他节点该节点已离开\(_left_\)。如果你强行杀死 agent 进程，其他集群成员将会检测到该节点失败\(_failed_\)。成员 left 后，它的服务和检查也将会从 catalog 中移除。成员失败后，健康检查会标记为崩溃，但并不会移除 catalog。Consul 会自动尝试重连 _faild_ 节点，以免在不稳定网络环境下失效；而 _left_ 的节点将不会再被联络。

另外，如果 agent 是 server 模式，将会通过[共识协议](https://consul.io/docs/internals/consensus.html)来避免潜在的服务中断。参见[进阶教程（二）](https://kingfree.gitbook.io/consul/jin-jie-jiao-cheng-er-jin-jie-cao-zuo)了解如何添加和移除服务。

