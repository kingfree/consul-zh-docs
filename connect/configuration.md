# Connect Configuration

There are many configuration options exposed for Connect. The only option that must be set is the "enabled" option on Consul Servers to enable Connect. All other configurations are optional and have reasonable defaults.

### Enable Connect on the Cluster <a id="enable-connect-on-the-cluster"></a>

The first step to use Connect is to enable Connect for your Consul cluster. By default, Connect is disabled. Enabling Connect requires changing the configuration of only your Consul _servers_ \(not client agents\). To enable Connect, add the following to a new or existing [server configuration file](https://www.consul.io/docs/agent/options.html). In HCL:

```text
connect {
  enabled = true
}
```

This will enable Connect and configure your Consul cluster to use the built-in certificate authority for creating and managing certificates. You may also configure Consul to use an external [certificate management system](https://www.consul.io/docs/connect/ca.html), such as [Vault](https://vaultproject.io/).

No agent-wide configuration is necessary for non-server agents. Services and proxies may always register with Connect settings, but they will fail to retrieve or verify any TLS certificates. This causes all Connect-based connection attempts to fail until Connect is enabled on the server agents.

**Note:** Connect is enabled by default when running Consul in dev mode with `consul agent -dev`.

**Security note:** Enabling Connect is enough to try the feature but doesn't automatically ensure complete security. Please read the [Connect production guide](https://www.consul.io/docs/guides/connect-production.html) to understand the additional steps needed for a secure deployment.

### Built-In Proxy Options <a id="built-in-proxy-options"></a>

This is a complete example of all the configuration options available for the built-in proxy. Note that only the `service.connect.proxy.config` and `service.connect.proxy.upsteams[].config` maps are being described here, the rest of the service definition is shown for context but is [described elsewhere](https://www.consul.io/docs/connect/proxies.html#managed-proxies).

```text
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

**Proxy Config Key Reference**

All fields are optional with a sane default.

* [`bind_address`](https://www.consul.io/docs/connect/configuration.html#bind_address) - The address the proxy will bind it's _public_ mTLS listener to. It defaults to the same address the agent binds to.
* [`bind_port`](https://www.consul.io/docs/connect/configuration.html#bind_port) - The port the proxy will bind it's _public_ mTLS listener to. If not provided, the agent will attempt to assign one from its [configured proxy port range](https://www.consul.io/docs/agent/options.html#proxy_min_port) if available. By default the range is \[20000, 20255\] and the port is selected at random from that range.
* [`tcp_check_address`](https://www.consul.io/docs/connect/configuration.html#tcp_check_address) - The address the agent will run a [TCP health check](https://www.consul.io/docs/agent/checks.html) against. By default this is the same as the proxy's [bind address](https://www.consul.io/docs/connect/configuration.html#bind_address) except if the bind\_address is `0.0.0.0` or `[::]` in which case this defaults to `127.0.0.1` and assumes the agent can dial the proxy over loopback. For more complex configurations where agent and proxy communicate over a bridge for example, this configuration can be used to specify a different _address_ \(but not port\) for the agent to use for health checks if it can't talk to the proxy over localhost or it's publicly advertised port. The check always uses the same port that the proxy is bound to.
* [`disable_tcp_check`](https://www.consul.io/docs/connect/configuration.html#disable_tcp_check) - If true, this disables a TCP check being setup for the proxy. Default is false.
* [`local_service_address`](https://www.consul.io/docs/connect/configuration.html#local_service_address) - The `[address]:port` that the proxy should use to connect to the local application instance. By default it assumes `127.0.0.1` as the address and takes the port from the service definition's `port` field. Note that allowing the application to listen on any non-loopback address may expose it externally and bypass Connect's access enforcement. It may be useful though to allow non-standard loopback addresses or where an alternative known-private IP is available for example when using internal networking between containers.
* [`local_connect_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#local_connect_timeout_ms) - The number of milliseconds the proxy will wait to establish a connection to the _local application_ before giving up. Defaults to `1000` or 1 second.
* [`handshake_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#handshake_timeout_ms) - The number of milliseconds the proxy will wait for _incoming_ mTLS connections to complete the TLS handshake. Defaults to `10000` or 10 seconds.
* [`upstreams`](https://www.consul.io/docs/connect/configuration.html#upstreams) - **Deprecated** Upstreams are now specified in the `connect.proxy` definition. Upstreams specified in the opaque config map here will continue to work for compatibility but it's strongly recommended that you move to using the higher level [upstream configuration](https://www.consul.io/docs/connect/proxies.html#upstream-configuration).

**Proxy Upstream Config Key Reference**

All fields are optional with a sane default.

* [`connect_timeout_ms`](https://www.consul.io/docs/connect/configuration.html#connect_timeout_ms) - The number of milliseconds the proxy will wait to establish a TLS connection to the discovered upstream instance before giving up. Defaults to `10000` or 10 seconds.

