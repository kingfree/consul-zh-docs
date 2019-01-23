# 2. Consul 参考架构

随着应用程序逐步向动态架构迁移，服务的扩展和服务间通信的管理变得越来越具有挑战性。Consul 的服务发现功能提供了应用之间的动态连接。Consul 还可以监控节点健康状态并确保只有健康的服务被发现使用。Consul 构建的运行时配置允许通过全局设施进行更新。

本文档提供了一套建议实现和参考架构，包括 Consul 生产环境部署的系统要求、数据中心设计、网络和性能优化。

### 架构依赖 <a id="infrastructure-requirements"></a>

#### Consul Server <a id="consul-servers"></a>

Consul server agent 负责维护集群状态、响应 RPC 查询（读取）以及处理所有写入操作。鉴于 Consul server agent 需要处理大部分繁重工作，服务器性能对于整个 Consul 集群就显得非常重要。

下面给出了一些高级服务器配置。建议不要使用共享 CPU。

| 类型 | CPU | 内存 | 磁盘 | 典型云示例 |
| :--- | :--- | :--- | :--- | :--- |
| 小型 | 2 core | 8-16 GB RAM | 50GB | **AWS**: m5.large, m5.xlarge |
| ​ | ​ | ​ | ​ | **Azure**: Standard\_A4\_v2, Standard\_A8\_v2 |
| ​ | ​ | ​ | ​ | **GCE**: n1-standard-8, n1-standard-16 |
| 大型 | 4-8 core | 32-64 GB RAM | 100GB | **AWS**: m5.2xlarge, m5.4xlarge |
| ​ | ​ | ​ | ​ | **Azure**: Standard\_D4\_v3, Standard\_D5\_v3 |
| ​ | ​ | ​ | ​ | **GCE**: n1-standard-32, n1-standard-64 |

**硬件性能考虑**

* 小型服务器用于大多数刚起步的生产环境部署和开发、测试环境。
* 大型服务器用于工作负载持续较高的生产环境。

{% hint style="warning" %}
**注意** 对于大型负载，还要确保磁盘的 IOPS 水平跟得上日志更新速度。
{% endhint %}

更多服务器依赖参见[服务器优化](https://www.consul.io/docs/guides/performance.html)文档。

### 架构图 <a id="infrastructure-diagram"></a>

​

![](https://learn.hashicorp.com/assets/images/consul-arch.png)

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

如果由于网络问题 agent 无法遍历全量网格，则可以使用 Consul 自己的[网络分段](https://www.consul.io/docs/enterprise/network-segments/index.html)。网络分段是 Consul Enterprise 的功能，允许在同一集群下为共享 Raft 服务器创建多个租户\(tenant\)。每个租户都有自己的交流池且不与池外的 agent 通信。但是 KV 存储则是在这些租户之间共享的。如果 Consul 王国分段不可用，则可以通过分别创建 [Consul 数据中心](https://kingfree.gitbook.io/consul/guides/datacenters)来实现 agent 之间的隔离。

#### 多数据中心 <a id="multiple-datacenters"></a>

运行相同服务的不同数据中心的 Consul 集群可以用外网连接起来。集群的行为是互相独立的，且只会通过外网的 8302 端口进行通信。除非通过 CLI 或 API 特别指明，默认情况 Consul 只会返回本地数据中心的结果。Consul 不会在多数据中心之间共享数据。[consul-replicate](https://github.com/hashicorp/consul-replicate) 工具可以用来定期同步 KV 数据。

{% hint style="info" %}
建议启用 TLS 服务器名以避免无意间的 agent 混连。
{% endhint %}

进阶可以参考 Consul Enterprise 的[网络分区](https://www.consul.io/api/operator/area.html)\(network area\)功能。

一个典型的用法是在数据中心1\(dc1\)主机中为其他数据中心共享一个类似 LDAP（或 ACL 数据中心）的服务。然而由于规则问题，dc2 中的服务器不能直接与 dc3 的服务器进行连接。这不能简单地通过基础外网服务进行连接。基本联结\(basic federation\)需要 dc1, dc2 和 dc3 的所有服务器都连接在一个完整的网格内，并开启了交流端口\(`8302 tcp/udp`\)和 RPC\(`8300`\)通信端口。

网络分区允许数据中心通过可发现的外网连接互相交流。使用网络分区，dc1 的服务器可以与 dc2 和 dc3 的服务器进行通信，而并不需要在 dc2 和 dc3 之间建立连接。网络分区中的服务仅会通过 RPC 进行通信，这取消了跨数据中心共享和维护交流协议的对称加密所产生的开销。它还使得针对于交流端口的攻击被安全网关或者防火墙阻挡在外。

Consul 的[预备查询](https://www.consul.io/api/query.html)\(prepared query\)可以使得服务发现支持数据中心故障转移。比如本地 dc1 的 `payment` 服务挂掉了，预备查询机制就会引导用户回到最近的数据中心去检查该服务的其他健康实例。

{% hint style="warning" %}
**提示** 要想跨数据中心执行预备查询，Consul 集群之间必须通过外网连接。
{% endhint %}

预备查询默认优先解决本地数据中心的查询。预备查询并不支持对 KV 存储的查询。预备查询支持 ACL。预备查询还用于在 Raft 中保持服务器执行的配置和模板的一致性。

### 网络连通性 <a id="network-connectivity"></a>

内网通信发生在但数据中心的所有 agent 之间，每个 agent 都会向成员列表的随机成员发送定期探测。无论 agent 运行在 server 模式还是 client 模式，都会参与这一交流。刚开始会每秒通过 UDP 进行，如果节点在 200ms 内未能完成确认，则会通过 TCP 进行探测。如果 TCP 超过 10s，它会从指定的数目的随机节点去探测这个节点（也成为间接探测）。如果仍然没有该节点的响应，就认为该 agent 已下线。

agent 的状态直接影响服务发现的结果。如果 agent 下线，上面监控的服务也会被标记为下线状态。

此外，agent 还定期通过 TCP 进行全量同步，这使得每个 agent 都能知道它周围的成员列表，包括节点的名字、IP 地址和健康状态。这些操作的开销取决于上面提到的交流协议，它由集群大小限制着同步速率以保持较低的负载。每次交流通常在 30 秒到 5 分钟之间。更多信息可参阅 [Serf Gossip 文档](https://www.serf.io/docs/internals/gossip.html)。

在跨 L2 的大型网络中，流量会穿过防火墙和路由器。ACL 和防火墙规则需要启用以下端口：

| 名称 | 端口 | 标识 | 描述 |
| :--- | :--- | :--- | :--- |
| Server RPC | 8300 | ​ | 用于接受来自其他 agent 的入站请求。仅 TCP。 |
| Serf LAN | 8301 | ​ | 用于在局域网上交流。所有 agent 都依赖此。TCP 和 UDP。 |
| Serf WAN | 8302 | `-1` 以禁用 \(Consul 1.0.7 可用\) | 用于在广域网上交流。TCP 和 UDP。 |
| HTTP API | 8500 | `-1` 以禁用 | 用于客户端与 HTTP API 通信。仅 TCP。 |
| DNS Interface | 8600 | `-1` 以禁用 | ​ |

{% hint style="info" %}
依前面数据中心设计一节所说，网络分区和网络分段可以避免不同子网之间去开启防火墙端口。
{% endhint %}

默认情况下 HTTP 和 DNS 端口只会监听本地地址。

