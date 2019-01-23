# 2. 部署指南

本节部署指南包含了安装和配置如上节参考架构所述的单个 Consul 集群方案。

虽然现实中的实践不一定尽如前述架构，但仍强烈建议使用类似的架构体系。例如在多台物理主机或者虚拟机（配好防火墙）上实现高可用性。

为提供高可用性单集群架构，我们建议按照该架构在集群中部署不止一个 Consul server。

![&#x67B6;&#x6784;&#x56FE;](https://learn.hashicorp.com/assets/images/consul-arch-single.png)

所有的 Consul 主机上都要完成以下步骤。

* 下载 Consul
* 安装 Consul
* 配置 systemd
* 配置 Consul \(server\) 或 \(client\)
* 启动 Consul

本手册适用于运行着 systemd 和服务管理器的类 Linux 主机。

### 下载 Consul

可以从 [https://releases.hashicorp.com/consul/](https://releases.hashicorp.com/consul/) 下载预编译好的二进制文件。企业版请从指定地址下载。

下载完成后校验一下压缩包完整性。官方提供了[校验方案](https://www.hashicorp.com/security.html)。

```text
CONSUL_VERSION="1.2.0"
curl --silent --remote-name https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip
curl --silent --remote-name https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_SHA256SUMS
curl --silent --remote-name https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_SHA256SUMS.sig
```

### 安装 Consul

解压已下载的包，移动 `consul` 到 `/usr/local/bin`，检查是否在系统路径可用。

```text
unzip consul_${CONSUL_VERSION}_linux_amd64.zip
sudo chown root:root consul
sudo mv consul /usr/local/bin/
consul --version
```

`consul` 命令支持参数、标识和子命令的自动补全，建议启用：

```text
consul -autocomplete-install
complete -C /usr/local/bin/consul consul
```

创建一个没有权限的系统用户来运行 Consul，并分配一个数据目录：

```text
sudo useradd --system --home /etc/consul.d --shell /bin/false consul
sudo mkdir --parents /opt/consul
sudo chown --recursive consul:consul /opt/consul
```

### 配置 systemd

Systemd 使用[文档中的默认设置](https://www.freedesktop.org/software/systemd/man/systemd.directives.html)，所以需要写配置文件来定义非默认服务。

创建一个配置文件：

```text
sudo touch /etc/systemd/system/consul.service
```

配置 Consul 服务：

```text
[Unit]
Description="HashiCorp Consul - A service mesh solution"
Documentation=https://www.consul.io/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/consul.d/consul.hcl

[Service]
User=consul
Group=consul
ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d/
ExecReload=/usr/local/bin/consul reload
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

下面是 `[Unit]` 节的参数说明：

* [`Description`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Description=) - 服务描述
* [`Documentation`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Documentation=) - 文档地址
* [`Requires`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Requires=) - 指定网络依赖
* [`After`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Before=) - 指定网络服务必须在该服务之前启动
* [`ConditionFileNotEmpty`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#ConditionArchitecture=) - 检查配置文件是否为空

下面是 `[Service]` 节的参数说明：

* [`User`, `Group`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#User=) - 以 consul 用户和组来运行 Consul
* [`ExecStart`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#ExecStart=) - 用 `agent` 命令启动并制定配置文件目录
* [`ExecReload`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#ExecReload=) - 服务重新加载的执行指令
* [`KillMode`](https://www.freedesktop.org/software/systemd/man/systemd.kill.html#KillMode=) - 把 Consul 当成单个进程
* [`Restart`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#RestartSec=) - 如果异常退出就重新启动进行
* [`LimitNOFILE`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#Process%20Properties) - 限制文件描述符数量

下面是 `[Install]` 节的参数说明：

* [`WantedBy`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#WantedBy=) - 为多用户运行级别创建弱依赖

### 配置 Consul \(server\)

Consul 使用[该文档的默认配置](https://www.consul.io/docs/agent/options.html)，自定义的配置可以从多个文件中读取，参见[文档](https://www.consul.io/docs/agent/options.html)了解配置是如何加载和覆盖默认值的。

Consul server agent 需要的配置是 client agent 的超集。我们用 `consul.hcl` 表示普通 Consul 配置文件，用 `server.hcl` 表示 server agent 的配置。

#### 通常配置

创建一个配置文件  `/etc/consul.d/consul.hcl`：

```text
sudo mkdir --parents /etc/consul.d
sudo touch /etc/consul.d/consul.hcl
sudo chown --recursive consul:consul /etc/consul.d
sudo chmod 640 /etc/consul.d/consul.hcl
```

向 `consul.hcl` 加入如下配置：

{% hint style="warning" %}
**提示** `datacenter` 参数标识该集群所在的数据中心。`encrypt` 参数用 `consul keygen` 的输出来替换，而且每台机器配置都要保持一致。
{% endhint %}

```text
datacenter = "dc1"
data_dir = "/opt/consul"
encrypt = "Luj2FZWwlt8475wD1WtwUQ=="
```

* [`datacenter`](https://www.consul.io/docs/agent/options.html#_datacenter) - agent 所在的数据中心。
* [`data_dir`](https://www.consul.io/docs/agent/options.html#_data_dir) - agent 存储状态的目录。
* [`encrypt`](https://www.consul.io/docs/agent/options.html#_encrypt) - Consul 网络通信的加密密钥。

#### 自动加入集群

`retry_join` 参数可以通过 DNS 地址、IP 地址或云机制，通过一个 Consul server配置集群里所有 Consul agent，而不需要手动加入这些节点了。

该参数可以写到 `consul.hcl` 配置里：

{% hint style="warning" %}
**提示** 可以用 DNS 地址、IP 地址或者云服务自动加入标识来配置 `retry_join` 参数。
{% endhint %}

```text
retry_join = ["172.16.0.11"]
```

* [`retry_join`](https://www.consul.io/docs/agent/options.html#retry-join) - 要加入的集群某个 agent 的地址。

#### Performance 参数

`performance` 参数允许调整不同 Consul 中子系统的性能。

添加到 `consul.hcl` 配置文件：

```text
performance {
  raft_multiplier = 1
}
```

* [`raft_multiplier`](https://www.consul.io/docs/agent/options.html#raft_multiplier) - Consul 控制 Raft 计时的伸缩因子。设置为 1 会使 Raft 运行在高性能模式（默认值是 0.7），建议用于生产环境。

更多关于 Raft 调优和 `raft_multiplier` 设置的信息，参见[文档](https://www.consul.io/docs/guides/performance.html)。

#### Telemetry 参数

`telemetry` 参数指定 Consul 如何度量上游系统。

如果你要改这个参数，请详细阅读[监控和指标指南](https://kingfree.gitbook.io/consul/day-1-operations/monitoring)。

#### Server 配置 <a id="server-configuration"></a>

创建配置文件 `/etc/consul.d/server.hcl`：

```text
sudo mkdir --parents /etc/consul.d
sudo touch /etc/consul.d/server.hcl
sudo chown --recursive consul:consul /etc/consul.d
sudo chmod 640 /etc/consul.d/server.hcl
```

按如下修改 `server.hcl`：

{% hint style="warning" %}
**提示** `bootstrap_expect` 是你要用的 server 数，推荐 3 或 5。
{% endhint %}

```
server = true
bootstrap_expect = 3
```

* [`server`](https://www.consul.io/docs/agent/options.html#_server) - 表示该 agent 运行在 server 模式还是 client  模式。
* [`bootstrap-expect`](https://www.consul.io/docs/agent/options.html#_bootstrap_expect) - 表示该数据中心有多少 server 节点。集群中的所有 server 节点的该字段都应一致。

#### Consul UI <a id="consul-ui"></a>

Consul 提供了基于 Web 的用户界面，可以在上面查看所有服务、节点和 Intention，比使用 CLI 或者 API 方便得多。

{% hint style="warning" %}
**提示** 你应该选择一台服务器来提供 Consul UI 服务。
{% endhint %}

将 UI 选项加入 `server.hcl` 即可启用 Consul UI：

```text
ui = true
```

### 配置 Consul \(client\)

Consul client agent 的配置其实是 server 的一个子集，参见前面配置一下 `consul.hcl` 就可以了。其他特殊配置，也只需要逐一写配置文件即可。

### 启动 Consul

使用 systemctl 来添加和启用 Consul 服务。也可以用 systemctl 查看 Consul 服务状态：

```text
sudo systemctl enable consul
sudo systemctl start consul
sudo systemctl status consul
```

### 总结

In this guide you configured servers and clients in accordance to the reference architecture. This is the first step in deploying your first datacenter. In the next guide, you will learn how to configure backups to ensure the cluster state is save encase of a failure situation.

To create a secure cluster, we recommend completing the [ACL bootstrap guide](https://learn.hashicorp.com/consul/advanced/day-1-operations/acl-guide), [agent encryption guide](https://learn.hashicorp.com/consul/advanced/day-1-operations/agent-encryption), and [certificates guide](https://learn.hashicorp.com/consul/advanced/day-1-operations/certificates). All three guides are in the Day 1 learning path.

Finally, we also recommend reviewing the [Windows agent guide](https://www.consul.io/docs/guides/windows-guide.html) and [Consul in containers guide](https://www.consul.io/docs/guides/consul-containers.html) for a mixed workload environment.

