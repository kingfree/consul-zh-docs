# Connect

Consul Connect 提供了一种通过 TLS 信道认证服务之间连接的方式。应用程序可以通过 [sidecar 代理](https://kingfree.gitbook.io/consul/connect/proxies)自动建立 TLS 连接，在无需知晓 Connect 存在的情况下接管出入流量。应用程序也可以通过[在内部集成 Connect](https://kingfree.gitbook.io/consul/connect/native) 来获取更好的性能和安全性。

Connect 使服务间加密和身份鉴权这一最佳部署方案得以到处可用。相对于传统的基于 IP 地址的主机间鉴权，Connect 使用 intention 来实现服务身份访问控制。这使得管理访问权限更为简单，让你可以自由地启用和移动某个服务，比如在 Kubernetes 和 Nomad 环境中。另外，intention 不依赖于底层网络，所以 Connect 可以在物理网络、云网络、软件定义网络\(SDN\)或者交叉网络中任意运行。

### 工作原理 <a id="how-it-works"></a>

Connect 核心基于 [mTLS](https://en.wikipedia.org/wiki/Mutual_authentication)。

Connect 为每个服务提供了一个以 TLS 证书编码的身份信息。该证书用于与其他服务建立和接受连接。该证书编码方式与 [SPIFFE X.509](https://github.com/spiffe/spiffe/blob/master/standards/X509-SVID.md) 兼容，借此可以接入任何支持 SPIFFE 的系统。

客户端服务会通过[公共 CA 协议](https://www.consul.io/api/connect/ca.html#list-ca-root-certificates)来验证对方服务证书。这很像一个典型的 HTTPS 浏览器连接。在此之上，客户端提供它自己的证书给服务端，如果连接握手成功，该连接将被认证和进行加密。

服务端也通过公共 CA 协议来验证客户端整数。验证完成后，服务端将会调用[认证 API](https://www.consul.io/api/agent/connect.html#authorize) 来检查 Consul intention 的设置。如果认证 API 返回成功，连接就会建立，否则就会拒绝。

Consul 提供了一套内置的方案来生成并分发证书，还内建了对 [Vault](https://www.consul.io/docs/connect/index.html#) 的支持。PKI 系统可以随意插拔并[扩展](https://www.consul.io/docs/connect/index.html#)到支持任意系统。

Connect 的所有 API 通常都要在数毫秒内响应，并对已有服务产生最小的影响。这是因为 Connect 相关 API 都是基于本地 Consul agent 的本地接口的，所有 [agent Connect 接入点](https://www.consul.io/api/agent/connect.html)都实现了本地缓存、后台更新和阻塞查询。多数 API 调用都会落在内存上并在数毫秒内返回。

### 开始学习 Connect <a id="getting-started-with-connect"></a>

在不同环境使用 Connect 有不同方案：

* 基础教程中的 [Connect 介绍](https://learn.hashicorp.com/consul/getting-started/connect)给出了在两个服务间通信的简单例子。
* [Envoy 教程](https://www.consul.io/docs/guides/connect-envoy.html)讲述了通过以 Envoy 为代理使用 Docker 运行组件的方式。
* [Kubernetes 文档](https://www.consul.io/docs/platform/k8s/run.html) 介绍了如何按照官方架构，从一个已有 Consul 和 Envoy 的空 Kubernetes 集群上自动配置应用程序代理的步骤。

### Agent 缓存和性能 <a id="agent-caching-and-performance"></a>

To enable microsecond-speed responses on [agent Connect API endpoints](https://www.consul.io/api/agent/connect.html), the Consul agent locally caches most Connect-related data and sets up background [blocking queries](https://www.consul.io/api/index.html#blocking-queries) against the server to update the cache in the background. This allows most API calls such as retrieving certificates or authorizing connections to use in-memory data and respond very quickly.

All data cached locally by the agent is populated on demand. Therefore, if Connect is not used at all, the cache does not store any data. On first request, the data is loaded from the server and cached. The set of data cached is: public CA root certificates, leaf certificates, and intentions. For leaf certificates and intentions, only data related to the service requested is cached, not the full set of data.

Further, the cache is partitioned by ACL token and datacenters. This is done to minimize the complexity of the cache and prevent bugs where an ACL token may see data it shouldn't from the cache. This results in higher memory usage for cached data since it is duplicated per ACL token, but with the benefit of simplicity and security.

With Connect enabled, you'll likely see increased memory usage by the local Consul agent. The total memory is dependent on the number of intentions related to the services registered with the agent accepting Connect-based connections. The other data \(leaf certificates and public CA certificates\) is a relatively fixed size per service. In most cases, the overhead per service should be relatively small: single digit kilobytes at most.

The cache does not evict entries due to memory pressure. If memory capacity is reached, the process will attempt to swap. If swap is disabled, the Consul agent may begin failing and eventually crash. Cache entries do have TTLs associated with them and will evict their entries if they're not used. Given a long period of inactivity \(3 days by default\), the cache will empty itself.

### 多数据中心 <a id="multi-datacenter"></a>

Using Connect for service-to-service communications across multiple datacenters requires Consul Enterprise.

With Open Source Consul, Connect may be enabled on multiple Consul datacenters, but only services within the same datacenter can establish Connect-based, Authenticated and Authorized connections. In this version, Certificate Authority configurations and intentions are both local to their respective datacenters; they are not replicated across datacenters.

Full multi-datacenter support for Connect is available in [Consul Enterprise](https://www.consul.io/docs/enterprise/connect-multi-datacenter/index.html).

