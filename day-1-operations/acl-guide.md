# 4. 权限控制列表 ACL

Consul 使用权限控制列表\(Access Control List, ACL\)实现 UI、API、CLI、服务和 agent 交流的安全性。交流和 RPC 的详细内容参见[该教程](https://kingfree.gitbook.io/consul/day-1-operations/agent-encryption)。要为你的集群实现安全性请先配置 ACL。

ACL 的核心思想是，通过将规则归类为策略\(policy\)，然后通过 token 来分配一条或多条策略。

本节要求 Consul 1.4 以上版本，建议阅读 [ACL 系统文档](https://www.consul.io/docs/agent/acl-system.html)。老版本请看[老的 ACL 文档](https://www.consul.io/docs/guides/acl-legacy.html)。

启用 ACL 系统需要多个步骤，我们将介绍几个必要的步骤。

1. 为所有 server 启用 ACL
2. 创建初始化 token
3. 创建 agent 策略
4. 创建 agent token
5. 将新 token 应用到各 server
6. 为 client 启用 ACL 并应用 agent token

文末还有一些辅助和可选的步骤。

### 第1步：为所有 Consul Server 启用 ACL

首先要在配置文件中启用 ACL。如下，我们设置默认策略是“deny”，表示我们运行在白名单模式；而下行政策是“extend-cache”，表示我们将在中断期间忽略 TTL。

```javascript
{
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "down_policy": "extend-cache"
  }
}
```

请重启 server 以应用新的配置。请一次只重启一台，成功之后再重启其他的。

如果 ACL 启用成功，我们将在 leader 的日志里看到下面的警告：

```text
2018/12/12 01:36:40 [INFO] acl: Created the anonymous token
2018/12/12 01:36:40 [INFO] consul: ACL bootstrap enabled
2018/12/12 01:36:41 [INFO] agent: Synced node info
2018/12/12 01:36:58 [WARN] agent: Coordinate update blocked by ACLs
2018/12/12 01:37:40 [INFO] acl: initializing acls
2018/12/12 01:37:40 [INFO] consul: Created ACL 'global-management' policy
```

如果没有看到 ACL bootstrap enabled、created the anonymous token 和创建 `global-management` 的消息，说明 ACL 没有被正确启用。

注意，我们现在启用了 ACL，需要一个 token 才能进行操作。我们在创建 master token 之前是无法做任何事情的。简单起见，接下来我们将一直使用这个 master token。

### 第2步：创建引导 Token

当 ACL 启动后我们就可以创建第一个 bootstrap token，这是一个有无限权限的管理 token。它会被加入状态存储中，并共享到所有相关 server 上。

```text
$ consul acl bootstrap
AccessorID: edcaacda-b6d0-1954-5939-b5aceaca7c9a
SecretID: 4411f091-a4c9-48e6-0884-1fcb092da1c8
Description: Bootstrap Token (Global Management)
Local: false
Create Time: 2018-12-06 18:03:23.742699239 +0000 UTC
Policies:
00000000-0000-0000-0000-000000000001 - global-management
```

执行命令的机器上可以看到如下日志：

```text
2018/12/11 15:30:23 [INFO] consul.acl: ACL bootstrap completed
2018/12/11 15:30:23 [DEBUG] http: Request PUT /v1/acl/bootstrap (2.347965ms) from=127.0.0.1:40566
```

由于 ACL 已经启用，我们需要用它来完成别的操作。比如在获取成员列表时也要传递该 token：

```text
$ consul members -token "4411f091-a4c9-48e6-0884-1fcb092da1c8"
Node  Address            Status  Type    Build  Protocol  DC  Segment
fox   172.20.20.10:8301  alive   server  1.4.0  2         kc  <all>
bear  172.20.20.11:8301  alive   server  1.4.0  2         kc  <all>
wolf  172.20.20.12:8301  alive   server  1.4.0  2         kc  <all>
```

除了可以在参数中传递，还可以设置环境变量：

```text
$ export CONSUL_HTTP_TOKEN=4411f091-a4c9-48e6-0884-1fcb092da1c8
```

该 bootstrap token 也可以写到 server 配置文件的 `master` token 字段。

注意，bootstrap token 只能创建一个，在创建 master token 之后该 token 就会被禁用。当 ACL 成功引导并启动后，可以通过 [ACL API](https://www.consul.io/api/acl/acl.html) 来管理这些 token。

### 第3步：创建 Agent Token 策略

在创建 token 之前，我们需要创建它的策略。一条策略\(policy\)是一系列规则\(rule\)的集合，可以指定细粒度的权限。参见 [ACL 规则定义文档](https://www.consul.io/docs/agent/acl-rules.html)获取更多信息。

{% code-tabs %}
{% code-tabs-item title="agent-policy.hcl" %}
```text
node_prefix "" {
   policy = "write"
}
service_prefix "" {
   policy = "read"
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

该策略允许所有节点可被注册和访问，允许所有服务可读。注意，不建议在生产中使用这个简单策略。最佳实践是对集群中每个节点配置精确的策略和 token。

只需要创建一条策略就可以应用到所有 server 上。如果你没有设置 `CONSUL_HTTP_TOKEN` 变量，请参考上文配置成 bootstrap token。

```text
$ consul acl policy create  -name "agent-token" -description "Agent Token Policy" -rules @agent-policy.hcl
ID:           5102b76c-6058-9fe7-82a4-315c353eb7f7
Name:         agent-policy
Description:  Agent Token Policy
Datacenters:
Rules:
node_prefix "" {
   policy = "write"
}

service_prefix "" {
   policy = "read"
}
```

返回的是我们新创建的策略，可以用于接下来要创建的 agent token 了。

### 第4步：创建 Agent Token

用刚创建的策略可以创建 agent token 了，我们可以在任一 server 上执行该操作。本文中，所有 agent 共享同一个 token。注意，`SecretID` 才是用于验证 API 和 CLI 命令的 token，而不是上面那个。

```text
$ consul acl token create -description "Agent Token" -policy-name "agent-token"
AccessorID:   499ab022-27f2-acb8-4e05-5a01fff3b1d1
SecretID:     da666809-98ca-0e94-a99c-893c4bf5f9eb
Description:  Agent Token
Local:        false
Create Time:  2018-10-19 14:23:40.816899 -0400 EDT
Policies:
   fcd68580-c566-2bd2-891f-336eadc02357 - agent-token
```

### 第5步：将 Agent Token 加到其他 Server

配置 server 的最后一步，就是通过配置文件把该 token 分配到各 Consul server 上，然后重启：

{% code-tabs %}
{% code-tabs-item title="acl.json" %}
```javascript
{
  "primary_datacenter": "dc1",
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "down_policy": "extend-cache",
    "tokens": {
      "agent": "da666809-98ca-0e94-a99c-893c4bf5f9eb"
    }
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

这里可以看到不再有警告日志了，而是会显示出节点正在同步：

```text
2018/12/11 15:34:20 [DEBUG] agent: Node info in sync
```

请务必在配置 client 之前确保各 server 配置正确。如有问题而不解决，只会导致无谓重复的工作和更多的问题。

#### 确保 ACL 系统配置正常

在配置 client 之前先要检查一下 server 的健康状态，看 leader 的编目即可：

```text
curl http://127.0.0.1:8500/v1/catalog/nodes -H 'x-consul-token: 4411f091-a4c9-48e6-0884-1fcb092da1c8'
[
    {
        "Address": "172.20.20.10",
        "CreateIndex": 7,
        "Datacenter": "kc",
        "ID": "881cfb69-2bcd-c2a9-d87c-cb79fc454df9",
        "Meta": {
            "consul-network-segment": ""
        },
        "ModifyIndex": 10,
        "Node": "fox",
        "TaggedAddresses": {
            "lan": "172.20.20.10",
            "wan": "172.20.20.10"
        }
    }
]
```

所有字段必须如期望所示。尤其要注意，如果 `TaggedAddresses` 是 null 的话就表明 ACL 配的绝对有问题。请看看各 server 的 Consul 日志来查明发生了什么问题吧。

### 第6步：为所有 Consul Client 启用 ACL

因为 ACL 措施也会影响 Consul client，我们也要写上配置文件并重启这些 client 以启用 ACL。我们可以使用与 server 相同的 ACL agent token。之所以能这么用，是因为我们没有指定节点或服务的前缀。

{% code-tabs %}
{% code-tabs-item title="acl.json" %}
```javascript
{
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "down_policy": "extend-cache",
    "tokens": {
      "agent": "da666809-98ca-0e94-a99c-893c4bf5f9eb"
    }
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

然后再次通过 `/catalog` 检查这些 agent 是否已正确配置。

### 可选的 ACL 配置

现在我们可以为 CLI、UI 和 所有节点都配置一个指定的 token 了。下面是一些可选的配置步骤，你也可以创建更多适合你环境的策略。

#### 配置匿名 Token \(可选\)

当执行 `consul acl bootstrap` 时实际上是创建了一个匿名 token，如果没有指定 token 就会用这个。本节我们会用新创建的策略更新已有的 token。

ACL agent 启动后，并没有设置任何策略，甚至 `consul members` 这种指令都会被默认的“deny”拒绝掉。

```text
$ consul members
```

我们不会看到错误，因为 ACL 已经过滤掉了所有输出。

如果指定一个 token 的话，就会看到所有节点列表，因为它有对于空 node 前缀的写权限，也就是拥有访问所有节点的权限：

```text
$ CONSUL_HTTP_TOKEN=4411f091-a4c9-48e6-0884-1fcb092da1c8 consul members
Node  Address            Status  Type    Build  Protocol  DC  Segment
fox   172.20.20.10:8301  alive   server  1.4.0  2         kc  <all>
bear  172.20.20.11:8301  alive   server  1.4.0  2         kc  <all>
wolf  172.20.20.12:8301  alive   server  1.4.0  2         kc  <all>
```

我们希望不用 token 就能在多数环境下看到所有节点。我们可以配置一个 Consul 策略，该策略可以在不用 token 的时候以匿名 token 的身份去访问节点列表。它和别的 ACL token 其实是一样的，只是 ID 为 `anonymous` 而已。这里的例子给匿名 token 以所有节点的读取权限：

```text
$ consul acl policy create -name 'list-all-nodes' -rules 'node_prefix "" { policy = "read" }'
ID:           e96d0a33-28b4-d0dd-9b3f-08301700ac72
Name:         list-all-nodes
Description:
Datacenters:
Rules:
node_prefix "" { policy = "read" }

$ consul acl token update -id 00000000-0000-0000-0000-000000000002 -policy-name list-all-nodes -description "Anonymous Token - Can List Nodes"
Token updated successfully.
AccessorID:   00000000-0000-0000-0000-000000000002
SecretID:     anonymous
Description:  Anonymous Token - Can List Nodes
Local:        false
Create Time:  0001-01-01 00:00:00 +0000 UTC
Hash:         ee4638968d9061647ac8c3c99e9d37bfdd2af4d1eaa07a7b5f80af0389460948
Create Index: 5
Modify Index: 38
Policies:
   e96d0a33-28b4-d0dd-9b3f-08301700ac72 - list-all-nodes
```

不指定 token 时，匿名 token 会被隐式地使用。所以直接运行 `consul members` 就可以获取节点列表了：

```text
$ consul members
Node  Address            Status  Type    Build  Protocol  DC Segment
fox   172.20.20.10:8301  alive   server  1.4.0  2         kc  <all>
bear  172.20.20.11:8301  alive   server  1.4.0  2         kc  <all>
wolf  172.20.20.12:8301  alive   server  1.4.0  2         kc  <all>
```

匿名 token 也可以用于 DNS 查询，因为 DNS 查询并没有传递 token 的地方。我们可以试一下：

```text
$ dig @127.0.0.1 -p 8600 consul.service.consul

; <<>> DiG 9.8.3-P1 <<>> @127.0.0.1 -p 8600 consul.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 9648
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;consul.service.consul.         IN      A

;; AUTHORITY SECTION:
consul.                 0       IN      SOA     ns.consul. postmaster.consul. 1499584110 3600 600 86400 0
```

出现了 `NXDOMAIN` 错误，因为该匿名 token 并没有访问“consul”服务的权限。来更新一下匿名 token 的策略吧：

```text
$ consul acl policy create -name 'service-consul-read' -rules 'service "consul" { policy = "read" }'
ID:           3c93f536-5748-2163-bb66-088d517273ba
Name:         service-consul-read
Description:
Datacenters:
Rules:
service "consul" { policy = "read" }

$ consul acl token update -id 00000000-0000-0000-0000-000000000002 --merge-policies -description "Anonymous Token - Can List Nodes" -policy-name service-consul-read
Token updated successfully.
AccessorID:   00000000-0000-0000-0000-000000000002
SecretID:     anonymous
Description:  Anonymous Token - Can List Nodes
Local:        false
Create Time:  0001-01-01 00:00:00 +0000 UTC
Hash:         2c641c4f73158ef6d62f6467c68d751fccd4db9df99b235373e25934f9bbd939
Create Index: 5
Modify Index: 43
Policies:
   e96d0a33-28b4-d0dd-9b3f-08301700ac72 - list-all-nodes
   3c93f536-5748-2163-bb66-088d517273ba - service-consul-read
```

应用新的策略之后就可以通过 DNS 查询了：

```text
$ dig @127.0.0.1 -p 8600 consul.service.consul

; <<>> DiG 9.8.3-P1 <<>> @127.0.0.1 -p 8600 consul.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46006
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;consul.service.consul.         IN      A

;; ANSWER SECTION:
consul.service.consul.  0       IN      A       127.0.0.1
```

下一节讲述了如何修改匿名 token 的办法。

#### 指定 Agent 的默认 Token \(可选\)

可以在配置文件中 `acl.tokens.default` 写上这个匿名 token。当请求指向特定的 Consul agent 时将会使用该默认 token，而不会直接为空地去请求了。

这个很像匿名 token，但其实完全不一样。比如，它允许对 agent 进行更细粒度的控制，甚至对于指定前缀的 KV 存储限制读写权限。

当然，在我们的例子里使用的 `acl.tokens.default` 其实和匿名 token 确实是比较像的（因为是同一个值）。

#### 创建用于 UI 的 Token \(可选\)

可以用 ACL 来限制 Consul UI 的权限。上面的一顿操作之后，你的 UI 就会用匿名 ACL token 并导致大部分功能不可用。我们推荐使用一个专门用于 UL 的 ACL token，可以保存在浏览器会话里来实现认证。

首先创建一个新策略：

```text
$ consul acl policy create -name "ui-policy" \
                           -description "Necessary permissions for UI functionality" \
                           -rules 'key "" { policy = "write" } node "" { policy = "read" } service "" { policy = "read" }'

ID:           9cb99b2b-3c20-81d4-a7c0-9ffdc2fbf08a
Name:         ui-policy
Description:  Necessary permissions for UI functionality
Datacenters:
Rules:
key "" { policy = "write" } node "" { policy = "read" } service "" { policy = "read" }
```

随之创建一个 token：

```text
$ consul acl token create -description "UI Token" -policy-name "ui-policy"

AccessorID:   56e605cf-a6f9-5f9d-5c08-a0e1323cf016
SecretID:     117842b6-6208-446a-0d1e-daf93854857d
Description:  UI Token
Local:        false
Create Time:  2018-10-19 14:55:44.254063 -0400 EDT
Policies:
   9cb99b2b-3c20-81d4-a7c0-9ffdc2fbf08a - ui-policy
```

可以在 “Setting” 页面写上这个 token 以访问（相当于登录）。

于是我们就拥有了一个全功能可以对 KV 读写的 UI。

### 总结

[ACL API](https://www.consul.io/api/acl/acl.html) 可以用于创建应用级别的 token，可以精确到节点，也可以按角色分类。参考 [ACL 规则](https://www.consul.io/docs/agent/acl-rules.html)获取更多细节。

#### 安全须知

本节的配置允许我们默认看到所有节点，而受限的只能发现“consul”服务。如果你的环境有更严格的安全需求，请参考以下建议：

1. 本文是用配置文件添加 token 的，这说明这个 token 存在了本地磁盘上。处于安全考虑，token 可以通过 [Consul CLI](https://www.consul.io/docs/commands/acl/acl-set-agent-token.html) 添加。当然，这会使得管理变得更麻烦了。
2. 建议每个节点对 `node` 的写权限仅限于它自己，`service` 读权限则只赋予给指定前缀的服务。
3.  [Anti-entropy](https://www.consul.io/docs/internals/anti-entropy.html) 同步需要 ACL agent token 有对所有要注册服务的 `service:write` 权限。建议为每个分别的服务提供不同的 token，以通过 API 进行注册；或者在[配置文件](https://www.consul.io/docs/agent/services.html)中更改。注意 `service:write` 权限需要假定服务的身份，以使得 Consul Connect 的 intention 只在对每个服务实例无法赋予 `service:write` 权限时的强制措施。更多信息参考 [Connect 安全文档](https://www.consul.io/docs/connect/security.html)。

