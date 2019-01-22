# 6. 注册健康检查

我们已经学会了如何简单地运行 Consul、添加节点和服务、查询节点和服务。本节，我们将继续学习如何为节点和服务添加健康检查。健康检查是服务发现的重要部分，可以一定程度避免服务瘫痪。

本节内容基于前面构建的 Consul 集群，请保证你现在运行着一个双节点集群。

### 定义检查

和服务一样，健康检查也可以通过提供[检查定义文件](https://consul.io/docs/agent/checks.html)或者通过 [HTTP API](https://consul.io/api/health.html) 进行注册。

我们仍然使用定义文件的通用办法。

在 Consul 0.9.0 和之后的版本中，agent 必须设置 `enable_script_checks` 为 true 才能启用脚本检查。

分别为两个节点创建一个配置文件：

```text
vagrant@n2:~$ echo '{"check": {"name": "ping",
  "args": ["ping", "-c1", "baidu.com"], "interval": "30s"}}' \
  >/etc/consul.d/ping.json

vagrant@n2:~$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80,
  "check": {"args": ["curl", "localhost"], "interval": "10s"}}}' \
  >/etc/consul.d/web.json
```

第一个定义添加了一个名为“ping”的主机级别检查，该检查每 30 秒运行一次 `ping -c1 baidu.com`。对于基于脚本 `sccript` 的健康检查，它的运行时用户和运行 Consul 进程的是同样的。如果退出代码 ≥2，检查会标记为 failing，服务会被标记为不健康；退出代码为 1 则会触发 warning 状态。参见 [`script` 检查](https://kingfree.gitbook.io/consul/agent/checks)。

第二个命令修改服务名“web”，添加了一个每 10 秒的 curl 验证请求来检查服务是否可用。和主机级别检查一样，如果脚本返回代码 ≥2，检查会标记为 failing，服务标记为不健康。

现在重启第二个 agent，用 `consul reload` 或 `SIGHUP` 重载配置，你会看到：

```text
==> Starting Consul agent...
...
    [INFO] agent: Synced service 'web'
    [INFO] agent: Synced check 'service:web'
    [INFO] agent: Synced check 'ping'
    [WARN] Check 'service:web' is now critical
```

前面几行表示服务已同步，最后一行表示 web 服务处于崩溃状态。因为我们没有真的运行一个 web 服务器，所以 curl 测试当然会失败。

### 检查健康状态

对于这两个简单的检查，可以使用 HTTP API 查看状态。首先来看看哪些服务崩溃了（当然，任何节点上执行都是可以的）：

```text
vagrant@n1:~$ curl http://localhost:8500/v1/health/state/critical
[
    {
        "Node": "agent-two",
        "CheckID": "service:web",
        "Name": "Service 'web' check",
        "Status": "critical",
        "Notes": "",
        "ServiceID": "web",
        "ServiceName": "web",
        "ServiceTags": [
            "rails"
        ]
    }
]
```

可以看到只有一个 web 服务处于 `critical` 状态。

另外，还可以用 DNS 来查询这个 web 服务。Consul 会返回空结果，因为服务处于不健康状态：

```text
dig @127.0.0.1 -p 8600 web.service.consul
...

;; QUESTION SECTION:
;web.service.consul.        IN  A
```

