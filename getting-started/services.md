# 3. 注册服务

上面讲了启动 agent 和查询集群成员和节点。下面我们将注册一个服务。

### 定义服务

服务可以通过[服务定义](https://www.consul.io/docs/agent/services.html)，或者调用 [HTTP API](https://www.consul.io/api/agent/service.html#register-service) 进行注册。

服务定义是最常用的注册服务方式，所以我们以此为例。

首先，创建 Consul 配置目录，Consul 会加载目录下所有配置文件。Unix 系统可以创建一个 `/etc/consul.d` 。

```text
$ mkdir ./consul.d
```

接着，编写服务定义配置文件。假设有一个名叫“web”服务运行在 80 端口，写入如下配置：

```text
$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' \
    > ./consul.d/web.json
```

现在，重启 agent，加上配置目录选项：

```text
$ consul agent -dev -config-dir=./consul.d
==> Starting Consul agent...
...
    [INFO] agent: Synced service 'web'
...
```

注意到输出中“已同步\(synced\)”了该服务，表示服务定义已从配置文件中读取，并成功注册进入服务编目。

如果你想注册多个服务， 可以在 Consul 配置目录中创建多个服务定义配置文件。

{% hint style="warning" %}
注意：在生产环境，你应当打开健康检查，并运行在 80 端口。简单起见，这里都没有做。
{% endhint %}

### 查询服务

当 agent 启动，服务同步成功后，就可以通过 DNS 或者 HTTP API 查询服务了。

#### DNS API

先用 DNS API 查吧。服务的 DNS 名是 `NAME.service.consul`。默认情况下，所有 DNS 名都在 `consul` 命名空间，也[可以配置](https://consul.io/docs/agent/options.html#domain)。`service` 子域告诉 Consul 我们要查询的是服务，`NAME` 则是服务的名字。

对于上面注册好的 Web 服务，会话查询结果如下：

```text
$ dig @127.0.0.1 -p 8600 web.service.consul

;; QUESTION SECTION:
;web.service.consul.        IN  A

;; ANSWER SECTION:
web.service.consul. 0   IN  A   127.0.0.1
```

如你所见，会返回一个 IP 地址为可用节点的`A` 记录，`A` 记录仅支持 IP 地址。

{% hint style="warning" %}
因为我们启动的是最小化的 `consul`，所以 `A` 记录返回的是 `127.0.0.1`。如果你想获取对于集群中其他节点有意义的结果，请使用 Consul agent 的`-advertise` 参数或者[服务定义](https://www.consul.io/docs/agent/services.html)中的`address` 字段。
{% endhint %}

你也可以通过 DNS API 查询包含“地址-端口”对的 `SRV` 记录：

```text
$ dig @127.0.0.1 -p 8600 web.service.consul SRV
;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 80 Armons-MacBook-Air.node.dc1.consul.

;; ADDITIONAL SECTION:
Armons-MacBook-Air.node.dc1.consul. 0 IN A  127.0.0.1
```

该 `SRV` 记录表示该 Web 服务运行在  `Armons-MacBook-Air.node.dc1.consul.` 的 80 端口，`A` 记录则作为附加部分返回。

最后，我们可以用标签来筛选服务，格式是 `TAG.NAME.service.consul`。例子如下，我们查询“rails”标签，就会得到以该标签注册的服务：

```text
$ dig @127.0.0.1 -p 8600 rails.web.service.consul
;; QUESTION SECTION:
;rails.web.service.consul.      IN  A

;; ANSWER SECTION:
rails.web.service.consul.   0   IN  A   127.0.0.1
```

#### HTTP API

作为 DNS API 的补充，HTTP API 也可以用于服务查询：

```text
$ curl http://localhost:8500/v1/catalog/service/web
[{"Node":"Armons-MacBook-Air","Address":"172.20.20.11","ServiceID":"web", \
    "ServiceName":"web","ServiceTags":["rails"],"ServicePort":80}]
```

编目\(catalog\) API 会给出可用服务列表。还可以进一步列出通过[健康检查](https://learn.hashicorp.com/consul/getting-started/checks)的健康实例列表，这也是 DNS 已经做了的：

```text
$ curl 'http://localhost:8500/v1/health/service/web?passing'
[{"Node":"Armons-MacBook-Air","Address":"172.20.20.11","Service":{ \
    "ID":"web", "Service":"web", "Tags":["rails"],"Port":80}, "Checks": ...}]
```

### 更新服务

修改配置文件将会更新服务定义，并发送 `SIGHUP` 信号给 agent。这使你的服务不至于下线或者不可用。

另外，HTTP API 可以动态地添加、移除和修改服务。

### 总结

其他服务定义可以参见 [API 文档](https://www.consul.io/api/agent/service.html)。

