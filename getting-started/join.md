# 5. Consul 集群

我们已经启动了一个 agent，并在进行了服务的注册和查询，还为服务间自动授权和加密连接配置了 Consul Connect。你可以看到 Consul 的使用是如何便利，但没有展示它是如何作为一个可扩展的生产级别的服务网格基础设施的。这一步我们将会创建一个包含多个成员的真正集群。

当 Consul agent 启动后，它并不知道其他节点的信息：它是一个孤立的集群。要想知道其他集群成员的信息，agent 需要加入\(_join_\)一个已存在的集群。为此，它只需要知道单个\(_single_\)已存在的成员。之后，agent 会与该成员交流并很快发现该集群中的其他成员。Consul agent 可以加入任意一个其他 agent，并不要求其运行在 server 模式。

### 启动多个 Agent

为了模拟真实的集群，我们用 [Vagrant](https://www.vagrantup.com/) 启动两个节点。配置文件可以参见[样例项目](https://github.com/hashicorp/consul/tree/master/demo/vagrant-cluster)。

首先启动两个节点：

```text
$ vagrant up
```

系统可用后，ssh 进去开始配置我们的集群。进入第一个节点：

```text
$ vagrant ssh n1
```

前面的例子里，我们用 [`-dev` 标识](https://consul.io/docs/agent/options.html#_dev)来快速启动了一个开发模式的 server。然而这并不适用于集群环境。自此，我们将省略 `-dev` 标记并换为下面所说的集群标识。

集群的每个节点都有一个唯一的名字。默认 Consul 会使用机器的主机名\(hostname\)，也可以通过 [`-node` 命令行选项](https://consul.io/docs/agent/options.html#_node)来手动指定。

我们可以指定一个绑定 [\`bind\` 地址](https://consul.io/docs/agent/options.html#_bind)：它是 Consul 的监听地址，并必须可以被其他 Consul 节点访问。但 bind 地址并不是必须的，但最好指定一个。Consul 默认会监听系统所有 IPv4 地址，但如果检测到多个私有 IP 地址则会报错。因为生产环境的服务器经常有多个网卡和地址，所以为了避免绑定在错误地址上你应该指定一个地址。

第一个节点成为了我们当前集群的唯一成员，用 [`server` 开关](https://consul.io/docs/agent/options.html#_server)表示。

[`-bootstrap-expect` 标识](https://consul.io/docs/agent/options.html#_bootstrap_expect)告诉 Consul server 我们希望有多少个节点加入进来。该标识的目的是防止在其他节点加入进来之前打印错误日志，参见[启动指南](https://kingfree.gitbook.io/consul/guides/bootstrapping)。

为了允许执行健康检查的脚本，我们设置 `-enable-script-checks` 标识为 `true`，后面会用到。生产中，你可能想要配置 [ACL](https://kingfree.gitbook.io/consul/day-1-operations/acl-guide) 来控制脚本的注册执行。

最后，加上 `config-dir` 标识，来指定服务和检查的定义配置文件位置。

把上面的参数加起来就像这样：

```text
vagrant@n1:~$ consul agent -server -bootstrap-expect=1 \
    -data-dir=/tmp/consul -node=agent-one -bind=172.20.20.10 \
    -enable-script-checks=true -config-dir=/etc/consul.d
...
```

好，让我们再开一个新的终端，连接到第二个节点：

```text
$ vagrant ssh n2
```

这次，我们按照 Vagrantfile 中配置的第二个节点的 IP 地址来指定 `bind` 地址，`node` 节点名为 `agent-two` ，且不运行在 server 模式下就不再开启 `server` 选项。

综上可以得到一条命令：

```text
vagrant@n2:~$ consul agent -data-dir=/tmp/consul -node=agent-two \
    -bind=172.20.20.11 -enable-script-checks=true -config-dir=/etc/consul.d
...
```

此时，你已经运行了两个 Consul agent：一个 server 和一个 client。两个 Consul agent 都不知道彼此的存在和相关信息，它们分属两个独立的单节点集群。你可以通过 `consul members` 来验证它们分别都只有一个成员。

### 加入集群

打开新终端，让第一个 agent 加入第二个 agent 里面：

```text
$ vagrant ssh n1
...
vagrant@n1:~$ consul join 172.20.20.11
Successfully joined cluster by contacting 1 nodes.
```

你可以看到两个 agent 都打印出了相关日志。再次运行 `consul members` 就可以看到两个节点了：

```text
vagrant@n2:~$ consul members
Node       Address            Status  Type    Build  Protocol
agent-two  172.20.20.11:8301  alive   client  0.5.0  2
agent-one  172.20.20.10:8301  alive   server  0.5.0  2
```

{% hint style="info" %}
**记住：**加入集群只需要知道一个_存在的成员_即可。加入集群后，agent 会自动和其他节点交流并获取全局的成员信息。
{% endhint %}

### 启动时自动加入集群

理想情况下，每当新节点加入你的数据中心，就应该自动加入 Consul 集群而不需人工干预。Consul 支持 通过指定标签来自动加入 AWS、Google Cloud 和 Azure 服务，可以参考[这里](https://www.consul.io/docs/agent/cloud-auto-join.html)学习如何配置。我们可以让一个新的节点不需任何硬编码配置即可加入集群。另外，你也可以用 `-join` 标识或者 `start_join` 设置项来硬编码其他已知的 Consul agent。

### 查询节点

和查询服务一样，Consul 也有 DNS API 和 HTTP API 用于查询节点。

DNS API 的查询名结构为 `NAME.node.consul` 或 `NAME.node.DATACENTER.consul`。如果忽略数据中心，Consul 只会查询本地数据中心。

例如，对于“agent-one”，我们可以查一下“agent-two”的地址：

```text
vagrant@n1:~$ dig @127.0.0.1 -p 8600 agent-two.node.consul
...

;; QUESTION SECTION:
;agent-two.node.consul. IN  A

;; ANSWER SECTION:
agent-two.node.consul.  0 IN    A   172.20.20.11
```

查询节点的功能对于系统管理任务是非常实用的。比如，通过查询服务运行在哪个地址来知道 SSH 到哪台机器。

### 离开集群

使用 `Ctrl-C` 平滑退出或者强制杀死 agent 都可以离开集群。平滑退出可以使得节点标记为 _left_ 状态，其他情况的退出则会标记为 _failed_。详见第二节。



