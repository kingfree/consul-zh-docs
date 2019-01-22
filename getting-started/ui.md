# 8. Web UI

Consul 提供了一个用户友好又使用的 Web UI，可以用于查看所有服务和节点、检查健康状态以及读写键值数据。这个 UI 自动支持多个数据中心。

要启动本机的 UI，就要在启动 Consul agent 时加上 `-ui` 参数：

```text
$ consul agent -dev -ui
...
```

UI 将会运行在 HTTP API 的 `/ui` 路径下。默认是 `http://localhost:8500/ui`。

可以用 [ACL](https://learn.hashicorp.com/consul/advanced/day-1-operations/acl-guide#create-tokens-for-ui-use-optional-) 控制访问 UI 的权限。如果[启用](https://learn.hashicorp.com/consul/advanced/day-1-operations/acl-guide)了 ACL，你可以在 UI 里限制用户查看和修改的权限。

[这里](http://demo.consul.io/)有一个现成的演示站点。

### 如何使用老的 UI

Consul 1.2.0 开始，原来的 Consul UI 就被废弃了。你仍然把环境变量  `CONSUL_UI_LEGACY` 设置为 `true` 来继续使用老版本的 UI。设置该环境变量为 false 或者不管它，就会使用最新版本的 UI 了。



