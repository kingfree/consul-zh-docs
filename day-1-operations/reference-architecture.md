# 2. Consul 参考架构

随着应用程序逐步向动态架构迁移，服务的扩展和服务间通信的管理变得越来越具有挑战性。Consul 的服务发现功能提供了应用之间的动态连接。Consul 还可以监控节点健康状态并确保只有健康的服务被发现使用。Consul 构建的运行时配置允许通过全局设施进行更新。

本文档提供了一套建议实现和参考架构，包括 Consul 生产环境不熟的系统要求、数据中心设计、网络和性能优化。

### 架构依赖 <a id="infrastructure-requirements"></a>

#### Consul Server <a id="consul-servers"></a>

Consul server agent 负责维护集群状态、响应 RPC 查询（读取）以及处理所有写入操作。鉴于 Consul server agent 需要处理大部分繁重工作，服务器性能对于整个 Consul 集群就显得非常重要。

下面给出了一些高级服务器配置。建议不要使用共享 CPU。

| 类型 | CPU | 内存 | 磁盘 | 典型云示例 |
| :--- | :--- | :--- | :--- | :--- |
| 小型 | 2 core | 8-16 GB RAM | 50GB | **AWS**: m5.large, m5.xlarge |
|  |  |  |  | **Azure**: Standard\_A4\_v2, Standard\_A8\_v2 |
|  |  |  |  | **GCE**: n1-standard-8, n1-standard-16 |
| 大型 | 4-8 core | 32-64 GB RAM | 100GB | **AWS**: m5.2xlarge, m5.4xlarge |
|  |  |  |  | **Azure**: Standard\_D4\_v3, Standard\_D5\_v3 |
|  |  |  |  | **GCE**: n1-standard-32, n1-standard-64 |

**硬件性能考虑**

* 小型服务器用于大多数刚起步的生产环境部署和开发、测试环境。
* 大型服务器用于工作负载持续较高的生产环境。

{% hint style="info" %}
**注意** 对于大型负载，还要确保磁盘的 IOPS 水平跟得上日志更新速度。
{% endhint %}

更多服务器依赖参见[服务器优化](https://www.consul.io/docs/guides/performance.html)文档。

### 架构图 <a id="infrastructure-diagram"></a>

![&#x53C2;&#x8003;&#x67B6;&#x6784;&#x56FE;](https://learn.hashicorp.com/assets/images/consul-arch.png)

### 数据中心设计 <a id="datacenter-design"></a>

一个典型的 Consul 集群包含三到五个 server 和若干 client，可以部署在一个物理的数据中心里，也可以分成多个数据中心。对于强读写的大型集群建议部署在同一个物理局域网下获取更好的性能。在云环境下，单数据中心可以跨越多个可用空间比如把每个服务分在单个主机上。Consul 还支持通过外网的多服务中心部署，其中的单个服务中心内部可能在相同的内网环境下。

#### 单数据中心 <a id="single-datacenter"></a>

建议把单 Consul 集群的应用部署在同一个服务中心里。Consul 支持将传统三层应用进行微服务化。

典型地，三到五个 server 是对性能和可用性的最好权衡。这些服务器基于 Raft 实现存储 catalog、session、预先查询、ACL 和 KV 更新。

建议单服务中心的最大节点数不要超过 5,000 个。对于强读写的集群，考虑到随机数的碰撞、键值对的大小和监控数量，这个最大节点数可能进一步减少。随着客户端的增加，服务器之间通过交流达成一致的收敛时间也会越来越长。类似地，新的 server 加入一个已有上千个节点的集群，会由于 KV 的大量更新塞爆日志。

{% hint style="info" %}
**提示** 对于强写集群，请考虑纵向扩展机器性能和低延迟的存储服务。
{% endhint %}

在集群上必须小心使用服务标签的查询。如果两个服务（比如 blue 和 green）运行在同一个集群下，必须用恰当的标签来区别它们。如果不用标签，那 blue 和 green 都会显示在查询结果之中。

如果由于网络问题 agent 无法遍历全量网格，则可以使用 Consul 自己的[网络分段](https://www.consul.io/docs/enterprise/network-segments/index.html)。这部分属于企业级功能，不再赘述。Network segments is a Consul Enterprise feature that allows the creation of multiple tenants which share Raft servers in the same cluster. Each tenant has its own gossip pool and doesn't communicate with the agents outside this pool. The KV store, however, is shared between all tenants. If Consul network segments cannot be used, isolation between agents can be accomplished by creating discrete [Consul datacenters](https://www.consul.io/docs/guides/datacenters.html).

#### 多数据中心 <a id="multiple-datacenters"></a>

Consul clusters in different datacenters running the same service can be joined by WAN links. The clusters operate independently and only communicate over the WAN on port `8302`. Unless explicitly configured via CLI or API, the Consul server will only return results from the local datacenter. Consul does not replicate data between multiple datacenters. The [consul-replicate](https://github.com/hashicorp/consul-replicate) tool can be used to replicate the KV data periodically.

A good practice is to enable TLS server name checking to avoid accidental cross-joining of agents.

Advanced federation can be achieved with the [network areas](https://www.consul.io/api/operator/area.html) feature in Consul Enterprise.

A typical use case is where datacenter1 \(dc1\) hosts share services like LDAP \(or ACL datacenter\) which are leveraged by all other datacenters. However, due to compliance issues, servers in dc2 must not connect with servers in dc3. This cannot be accomplished with the basic WAN federation. Basic federation requires that all the servers in dc1, dc2 and dc3 are connected in a full mesh and opens both gossip \(`8302 tcp/udp`\) and RPC \(`8300`\) ports for communication.

Network areas allows peering between datacenters to make the services discoverable over WAN. With network areas, servers in dc1 can communicate with those in dc2 and dc3. However, no connectivity needs to be established between dc2 and dc3 which meets the compliance requirement of the organization in this use case. Servers that are part of the network area communicate over RPC only. This removes the overhead of sharing and maintaining the symmetric key used by the gossip protocol across datacenters. It also reduces the attack surface at the gossip ports since they no longer need to be opened in security gateways or firewalls.

Consul's [prepared queries](https://www.consul.io/api/query.html) allow clients to do a datacenter failover for service discovery. For example, if a service `payment` in the local datacenter dc1 goes down, a prepared query lets users define a geographic fallback order to the nearest datacenter to check for healthy instances of the same service.

**NOTE** Consul clusters must be WAN linked for a prepared query to work across datacenters.

Prepared queries, by default, resolve the query in the local datacenter first. Querying KV store features is not supported by the prepared query. Prepared queries work with ACL. Prepared query config/templates are maintained consistently in Raft and are executed on the servers.

### 网络连通性 <a id="network-connectivity"></a>

LAN gossip occurs between all agents in a single datacenter with each agent sending a periodic probe to random agents from its member list. Agents run in either client or server mode, both participate in the gossip. The initial probe is sent over UDP every second. If a node fails to acknowledge within `200ms`, the agent pings over TCP. If the TCP probe fails \(10 second timeout\), it asks configurable number of random nodes to probe the same node \(also known as an indirect probe\). If there is no response from the peers regarding the status of the node, that agent is marked as down.

The agent's status directly affects the service discovery results. If an agent is down, the services it is monitoring will also be marked as down.

In addition, the agent also periodically performs a full state sync over TCP which gossips each agentâs understanding of the member list around it \(node names, IP addresses, and health status\). These operations are expensive relative to the standard gossip protocol mentioned above and are synced at a rate determined by cluster size to keep overhead low. It's typically between 30 seconds and 5 minutes. For more details, refer to [Serf Gossip docs](https://www.serf.io/docs/internals/gossip.html)

In a larger network that spans L2 segments, traffic typically traverses through a firewall and/or a router. ACL or firewall rules must be updated to allow the following ports:

| 名称 | 端口 | 标识 | 描述 |
| :--- | :--- | :--- | :--- |
| Server RPC | 8300 |  | 用于接受来自其他 agent 的入站请求。仅 TCP。 |
| Serf LAN | 8301 |  | 用于在局域网上交流。所有 agent 都依赖此。TCP 和 UDP。 |
| Serf WAN | 8302 | `-1` 以禁用 \(Consul 1.0.7 可用\) | 用于在广域网上交流。TCP 和 UDP。 |
| HTTP API | 8500 | `-1` 以禁用 | 用于客户端与 HTTP API  通信。仅 TCP。 |
| DNS Interface | 8600 | `-1` 以禁用 |  |

As mentioned in the [datacenter design section](https://learn.hashicorp.com/consul/advanced/day-1-operations/reference-architecture#datacenter-design), network areas and network segments can be used to prevent opening up firewall ports between different subnets.

By default agents will only listen for HTTP and DNS traffic on the local interface.

### 总结 <a id="summary"></a>

Next, review the Deployment Guide to learn the steps required to install and configure a single HashiCorp Consul cluster.

