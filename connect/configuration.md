# Connect 配置

有很多可以暴露给 Connect 的设置项，除了需要启用“enabled”选项之外，其他选项都有各自合理的默认值。

### 为集群启用 Connect <a id="enable-connect-on-the-cluster"></a>

首先应该为集群启用 Connect。默认情况 Connect 是禁用的。要想启用，只需要修改 Consul server（而非 client）设置，创建一个新的或修改已有的 HCL 文件如下：

```text
connect {
  enabled = true
}
```

该操作会启用 Connect 并用内置的证书认证机制来配置你的 Consul 集群。你也可以用 Vault 这样的工具来扩展 Consul 的[认证管理系统](https://kingfree.gitbook.io/consul/connect/ca)。

不提供代理服务的 agent 不需要配置该选项。任意配置的话会导致服务和代理会用 Connect 的设置来验证 TLS 证书而注册失败，直到 server 也配置上了 Connect。

{% hint style="info" %}
**提示** 开发模式下 `consul agent -dev` 会默认启用 Connect。
{% endhint %}

{% hint style="info" %}
安全提示 仅启用 Connect 的话对于体验功能是可以的，但不保证完全安全。请阅读 [Connect 生产指南](https://kingfree.gitbook.io/consul/guides/connect-production)来学习安全部署需要的更多步骤。
{% endhint %}

### 内建代理选项 <a id="built-in-proxy-options"></a>

这是内建代理可用的完整配置项，需要配置到服务定义文件里。注意这里只有 `service.connect.proxy.config` 和 `service.connect.proxy.upsteams[].config` ，其他服务定义的部分请参见[别的描述](https://kingfree.gitbook.io/consul/connect/proxies.html#managed-proxies)。

```javascript
{
  "service": {
    ...
    "connect": {
      "proxy": {
        "config": {
          "bind_address": "0.0.0.0",
          "bind_port": 20000,
          "tcp_check_address": "192.168.0.1",
          "disable_tcp_check": false,
          "local_service_address": "127.0.0.1:1234",
          "local_connect_timeout_ms": 1000,
          "handshake_timeout_ms": 10000,
          "upstreams": [...]
        },
        "upstreams": [
          {
            ...
            "config": {
              "connect_timeout_ms": 1000
            }
          }
        ]
      }
    }
  }
}
```

**Proxy Config 参数说明**

所有参数都有一个合理的默认值。

* [`bind_address`](https://www.consul.io/docs/connect/configuration.html#bind_address) - 代理绑定的_公共_ mTLS 地址。默认与 agent 的绑定地址相同。
* [`bind_port`](https://www.consul.io/docs/connect/configuration.html#bind_port) - 代理绑定的_公共_ mTLS 端口。 如果不填，agent 会从[代理端口范围](https://www.consul.io/docs/agent/options.html#proxy_min_port)中随机选一个出来，默认是 \[20000, 20255\]。
* [`tcp_check_address`](https://www.consul.io/docs/connect/configuration.html#tcp_check_address) - The address the agent will run a [TCP health check](https://www.consul.io/docs/agent/checks.html) against. By default this is the same as the proxy's [bind address](https://www.consul.io/docs/connect/configuration.html#bind_address) except if the bind\_address is `0.0.0.0` or `[::]` in which case this defaults to `127.0.0.1` and assumes the agent can dial the proxy over loopback. For more complex configurations where agent and proxy communicate over a bridge for example, this configuration can be used to specify a different _address_ \(but not port\) for the agent to use for health checks if it can't talk to the proxy over localhost or it's publicly advertised port. The check always uses the same port that the proxy is bound to.
* [`disable_tcp_check`](https://www.consul.io/docs/connect/configuration.html#disable_tcp_check) - If true, this disables a TCP check being setup for the proxy. Default is false.
* [`local_service_address`](https://www.consul.io/docs/connect/configuration.html#local_service_address) - The `[address]:port` that the proxy should use to connect to the local application instance. By default it assumes `127.0.0.1` as the address and takes the port from the service definition's `port` field. Note that allowing the application to listen on any non-loopback address may expose it externally and bypass Connect's access enforcement. It may be useful though to allow non-standard loopback addresses or where an alternative known-private IP is available for example when using internal networking between containers.
* [`local_connect_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#local_connect_timeout_ms) - The number of milliseconds the proxy will wait to establish a connection to the _local application_ before giving up. Defaults to `1000` or 1 second.
* [`handshake_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#handshake_timeout_ms) - The number of milliseconds the proxy will wait for _incoming_ mTLS connections to complete the TLS handshake. Defaults to `10000` or 10 seconds.
* [`upstreams`](https://www.consul.io/docs/connect/configuration.html#upstreams) - **已废弃** Upstreams are now specified in the `connect.proxy` definition. Upstreams specified in the opaque config map here will continue to work for compatibility but it's strongly recommended that you move to using the higher level [upstream configuration](https://www.consul.io/docs/connect/proxies.html#upstream-configuration).

**Proxy Upstream Config 参数说明**

所有参数都有一个合理的默认值。

* [`connect_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#connect_timeout_ms) - The number of milliseconds the proxy will wait to establish a TLS connection to the discovered upstream instance before giving up. Defaults to `10000` or 10 seconds.

