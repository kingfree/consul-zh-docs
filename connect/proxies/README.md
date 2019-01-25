# 代理

由 Connect 提供的代理可以让应用程序不做修改即可接入 Connect。每个服务的代理 sidecar 都透明地接管了服务的出入站连接，提供对 TLS 的自动包装和认证功能。

使用代理时，应该只让真正的服务接受本地回环地址上的连接。所有的外部连接都应该通过 Connect 协议提供的身份认证和鉴权机制建立。

{% hint style="info" %}
**废弃说明：**Managed Proxy 从 Consul 1.3 已废弃。参见 [managed proxy deprecation](https://www.consul.io/docs/connect/proxies/managed-deprecated.html) 获取更多信息。强烈建议使用旧版本的用户尽快迁移到本文所提供的解决方案。
{% endhint %}

### 代理服务定义 <a id="proxy-service-definitions"></a>

Connect 代理可以通过配置文件或者 API 进行[服务定义](https://www.consul.io/docs/agent/services.html)，本质上和其他服务是等同的。

另外，为了减少写很多配置的麻烦步骤，应用程序也可以通过内联 [sidecar 服务注册](https://www.consul.io/docs/connect/proxies/sidecar-service.html)这种简写方案来进行注册。

为启用 Connect 代理功能，还需要提供该代理要代表的服务信息。

描述一个代理服务至少要包含如下字段：

* [`kind`](https://www.consul.io/docs/connect/proxies.html#kind) \(string\) 必须。必须设为 `connect-proxy`。表示这是一个代理服务。
* [`proxy.destination_service_name`](https://www.consul.io/docs/connect/proxies.html#proxy-destination_service_name) \(string\) 必须。要代理的服务名。在 1.2.0 到 1.3.0 版本里该字段为 `proxy_destination`。
* [`port`](https://www.consul.io/docs/connect/proxies.html#port) 必须。其他 Connect 可以发现的地址。可选参数 `address` 默认会继承节点的地址。

最小样例：

```javascript
{
  "name": "redis-proxy",
  "kind": "connect-proxy",
  "proxy": {
    "destination_service_name": "redis"
  },
  "port": 8181
}
```

服务注册完成后，支持 Connect 的 client 就可以在查找 “redis” 时找到该代理。

#### Sidecar Proxy 选项说明 <a id="sidecar-proxy-fields"></a>

多数 Connect 代理都以“sidecar”模式部署，它们与其所代表的单个服务实例处在同一位置，并代理所有入站流量。这种情况下还需要设置如下字段：

* [`proxy.destination_service_id`](https://www.consul.io/docs/connect/proxies.html#proxy-destination_service_id) \(string\) 设为指定要代理的服务实例 _id_（如果 _name_ 不同）。代理后的服务会分配注册到同一个 agent 上，即便是未经验证的注册。
* [`proxy.local_service_port`](https://www.consul.io/docs/connect/proxies.html#proxy-local_service_port) \(string\) 需要指定代理连接到_本地_服务实例的端口。
* [`proxy.local_service_address`](https://www.consul.io/docs/connect/proxies.html#proxy-local_service_address) \(string\) 设为代理要连接到的本地服务的 IP 或主机名。默认为 `127.0.0.1`。

#### 完整配置样例 <a id="complete-configuration-example"></a>

下面展示了一个完整的代理注册实例。

```javascript
{
  "name": "redis-proxy",
  "kind": "connect-proxy",
  "proxy": {
    "destination_service_name": "redis",
    "destination_service_id": "redis1",
    "local_service_address": "127.0.0.1",
    "local_service_port": 9090,
    "config": {},
    "upstreams": []
  },
  "port": 8181
}
```

{% hint style="info" %}
**废弃说明：**从 1.2.0 到 1.3.0 版本，代理目标会在顶级用 `proxy_destination` 指定，该选项会支持到至少 1.5.0 版本，但强烈建议用新的`proxy.destination_service_name`。
{% endhint %}

**Proxy 参数**

* [`destination_service_name`](https://www.consul.io/docs/connect/proxies.html#destination_service_name) `string: <必须>` - 指定该实例代理服务的 _name_ 。包括 side-car 和中央负载均衡代理都需要指定该字段。它用于服务发现时根据给定的服务名来找到正确的代理实例。
* [`destination_service_id`](https://www.consul.io/docs/connect/proxies.html#destination_service_id) `string: <可选>` - 代理表示的单个指定服务实例 _ID_。仅对在同一节点上运行的 sidecar 代理有效，因为通过同一 Consul agent 注册的服务实例虽然节点一样但 ID 是唯一的。这对于在工具中显示哪个代理实例才是应用程序实例的 sider 很实用，并可以支持对代理粒度的细分。
* [`local_service_address`](https://www.consul.io/docs/connect/proxies.html#local_service_address) `string: <可选>` - 指定 sidecar 代理要连接到的本地实例地址。默认为 127.0.0.1。
* [`local_service_port`](https://www.consul.io/docs/connect/proxies.html#local_service_port) `int: <可选>` - 指定 sidecar 代理要连接到的本地应用程序实例的端口。默认是服务定义中对应 `destination_service_id` 定义的端口，否则响应为空。
* [`config`](https://www.consul.io/docs/connect/proxies.html#config) `object: <可选>` - 在将来的 API 调用中可能会用到的影响服务实例存储和返回的 JSON 配置对象。
* [`upstreams`](https://www.consul.io/docs/connect/proxies.html#upstreams) `array<Upstream>: <可选>` - 指定代理要为哪些 upstream 服务创建监听器。格式定义参见 [Upstream 配置参考](https://www.consul.io/docs/connect/proxies.html#upstream-configuration-reference)。

#### Upstream 选项说明 <a id="upstream-configuration-reference"></a>

下面的例子展示了可能的上游配置参数。

注意在 1.2.0 到 1.3.0 版本中，managed proxy upstream 都在 `connect.proxy.config`映射中定义，该格式几乎没做变化，但现在 managed proxy upstream 定义在了`connect.proxy.upstreams`里。老的地方已经弃用了，并在未来几个版本中会被自动转换为新的。两个 upstream 定义中唯一的不同在于  `destination_datacenter` 改成了 `datacenter`。

注意 `snake_case` 在[配置文件和 API 注册](https://www.consul.io/docs/agent/services.html#service-definition-parameter-case)中都可用。

```javascript
{
  "destination_type": "service",
  "destination_name": "redis",
  "datacenter": "dc1",
  "local_bind_address": "127.0.0.1",
  "local_bind_port": 1234,
  "config": {}
},
```

* [`destination_name`](https://www.consul.io/docs/connect/proxies.html#destination_name) `string: <必须>` - 指定服务名，或者要路由到的预备查询。
* [`local_bind_port`](https://www.consul.io/docs/connect/proxies.html#local_bind_port) `int: <必须>` - 指定该 upstream 为应用程序建立出站连接的本地监听端口。
* [`local_bind_address`](https://www.consul.io/docs/connect/proxies.html#local_bind_address) `string: <可选>` - 指定该 upstream 为应用程序建立出站连接的本地监听地址。默认为 `127.0.0.1`。
* [`destination_type`](https://www.consul.io/docs/connect/proxies.html#destination_type) `string: <可选>` - 指定查找实例所要连接的发现查询的类型。可用指为  `service` 或 `prepared_query`。默认为 `service`。
* [`datacenter`](https://www.consul.io/docs/connect/proxies.html#datacenter) `string: <可选>` - 指定发现查询的数据中心。默认为本地数据中心。
* [`config`](https://www.consul.io/docs/connect/proxies.html#config-1) `object: <可选>` - 指定该 upstream 提供给代理实例的额外配置。包含一个 JSON 对象，可以对给定的 upstream 配置超时或重试等选项。参见[内置代理配置参考](https://www.consul.io/docs/connect/configuration.html#built-in-proxy-options)文档，或参考 [Envoy 配置参考](https://www.consul.io/docs/connect/configuration.html#envoy-options)文档。

#### 动态 Upstream <a id="dynamic-upstreams"></a>

If an application requires dynamic dependencies that are only available at runtime, it must currently [natively integrate](https://www.consul.io/docs/connect/native.html)with Connect. After natively integrating, the HTTP API or [DNS interface](https://www.consul.io/docs/agent/dns.html#connect-capable-service-lookups) can be used.

