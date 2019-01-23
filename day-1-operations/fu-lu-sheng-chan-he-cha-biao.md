# 附录：生产核查表

该列表可以帮你部署第一个数据中心。该核查表并非详尽，你可能需要为你的环境添加别的任务。

### 架构规划

* 复习架构图
* 复习架构依赖

#### 端口

参见 [API 文档](https://kingfree.gitbook.io/consul/agent/options)配置这些端口。

* DNS 服务器
* HTTP API
* HTTPS API
* gRPC API
* Serf LAN 端口
* Serf WAN 端口
* Server RPC 地址
* 用于 sidecar 服务的最小端口号
* 用于 sidecar 服务的最大端口号

### 部署

#### Consul Server

* 阅读最新版本的 Consul [发行注解](https://www.consul.io/docs/upgrade-specific.html)。
* 下载 [Consul 二进制文件](https://www.consul.io/downloads.html)到所有服务器上。
* 自定义[服务器配置](https://www.consul.io/docs/agent/options.html)。
* 配置或禁用[自动化](https://learn.hashicorp.com/consul/day-2-operations/advanced-operations/autopilot)。
* 启用 RPC 通信的 [TLS](https://learn.hashicorp.com/consul/advanced/day-1-operations/agent-encryption#tls-encryption-for-rpc)。
* 启用[交流加密](https://learn.hashicorp.com/consul/advanced/day-1-operations/agent-encryption#gossip-encryption)。
* 启用 [Telemetry](https://learn.hashicorp.com/consul/advanced/day-1-operations/monitoring#enable-telemetry)。

#### Consul Client

* 
### 网络



### 安全



### 监控



### 错误恢复



