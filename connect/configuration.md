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
* [`tcp_check_address`](https://www.consul.io/docs/connect/configuration.html#tcp_check_address) - 运行 [TCP 健康检查](https://www.consul.io/docs/agent/checks.html)的地址。默认与代理的[绑定地址](https://www.consul.io/docs/connect/configuration.html#bind_address)一致，如果 bind\_address 是 `0.0.0.0` 或`[::]` 则会设置为 `127.0.0.1` 以使代理可以通过本地回环连接。如果 agent 和代理之间通过桥而无法通过 localhost 或建议的公共端口通信的话，该选项需要为该 agent 提供一个不同的_地址_（不是端口）用于健康检查。该检查的端口总是和代理绑定的端口一致。
* [`disable_tcp_check`](https://www.consul.io/docs/connect/configuration.html#disable_tcp_check) - 如果为 true 则会禁用代理的 TCP 检查。默认为 false。
* [`local_service_address`](https://www.consul.io/docs/connect/configuration.html#local_service_address) - 代理用于连接到本地应用程序实例的 `[address]:port` 地址。默认地址为 `127.0.0.1` ，端口是服务定义文件中的 `port` 字段。注意当应用程序可以监听非本地回环地址时，该选项可以绕过 Connect 而从公网访问。它可用通过非标准的本地回环地址，或者像在容器间内网这种的自定义私网 IP 可用地址。
* [`local_connect_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#local_connect_timeout_ms) - 代理建立到_本地应用_的超时等待毫秒数。默认为 `1000` 即 1 秒。
* [`handshake_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#handshake_timeout_ms) - 代理建立 TLS 握手等待 mTLS _入站流量_的超时时间。默认为 `10000` 即 10 秒。
* [`upstreams`](https://www.consul.io/docs/connect/configuration.html#upstreams) - **已废弃** Upstreams are now specified in the `connect.proxy` definition. Upstreams specified in the opaque config map here will continue to work for compatibility but it's strongly recommended that you move to using the higher level [upstream configuration](https://www.consul.io/docs/connect/proxies.html#upstream-configuration).

**Proxy Upstream Config 参数说明**

所有参数都有一个合理的默认值。

* [`connect_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#connect_timeout_ms) - 代理等待建立 TLS 连接到已发现的上游实例的超时等待时间。默认为 `10000` 即 10 秒。

