# 4. Connect

Consul 提供名为 **Connect** 的功能，实现了通过加密 TLS 的自动连接，以及授权服务间互相访问的机制。

应用程序不必修改为都去使用 Connect，[代理](https://consul.io/docs/connect/proxies.html)可以用于自动建立 TLS 连接而不依赖 Connect 来处理上下行流量。应用程序也可以[内部集成 Connect](https://consul.io/docs/connect/native.html) 以获取更好的性能和安全性。

{% hint style="info" %}
**安全须知：**基础教程旨在展示 Connect 功能并关注于开发环境模式的用法。这里不会讲 Connec 在生产环境的安全用法。请阅读[生产环境的 Connect 指南](https://kingfree.gitbook.io/consul/guides/connect-production)来权衡。
{% endhint %}

### 启动无感知 Connect 服务

首先启动一个对 Connect 无感知的服务。简易起见，这里使用 `socat` 的基础回显服务。服务接受 TCP 连接并显示回去任意发过来的数据。你可以通过包管理器轻松获取到 `socat`。

```text
$ socat -v tcp-l:8181,fork exec:"/bin/cat"
```

你可以用 `nc` 直接连上它来验证是否正常。一旦连上，就可以打什么出什么：

```text
$ nc 127.0.0.1 8181
hello
hello
echo
echo
```

`socat` 是老式 Unix 工具，我们的进程只支持基本的 TCP 连接。它没有加密，更不支持 TLS。这代表了你的数据中心中例如数据库、后台 Web 服务这些已有服务。

### 用 Consul 和 Connect 注册服务

下面，我们用 Consul 注册，这需要写个新的服务定义。和前面一样，除了额外定义的 Connect：

```text
$ cat <<EOF | sudo tee ./consul.d/socat.json
{
  "service": {
    "name": "socat",
    "port": 8181,
    "connect": { "sidecar_service": {} }
  }
}
EOF
```

保存后执行 `consul reload` 或发送 `SIGHUP` 信号到 Consul 以便读取新的配置文件。

注意第 6 行的不同，`"connet"` 表示的是要在 Consul 为该进程注册一个辅助代理\(sidecar proxy\)。代理进程表示服务，它从一个动态分配的端口接受下行连接，通过 TLS 鉴权，并代理到标准的 TCP 连接上。

这里的 sidebar 服务注册只告诉了 Consul 需要运行代理，Consul 不会真的去运行一个代理进程。

我们需要在另一个终端里启动该代理进程：

```text
$ consul connect proxy -sidecar-for socat
==> Consul Connect proxy starting...
    Configuration mode: Agent API
        Sidecar for ID: socat
              Proxy ID: socat-sidecar-proxy

...
```

### 连接到服务

接着，连接到服务。首先直接用 `consul connect proxy` 命令来配置并运行一个能代表服务的本地代理。它提供了对任何服务的装饰手法，便于通过 Connect 建立和其他服务的连接，这在开发环境下是个有用的工具。

下面的命令启动一个表示 “web” 的代理，我们使前面注册的 “socat” 上游依赖\(upstream dependency\)运行在 9191 端口，作为一个适配 Connect 的接入点，为 “web” 服务建立相互的 TLS 连接。

```text
$ consul connect proxy -service web -upstream socat:9191
==> Consul Connect proxy starting...
    Configuration mode: Flags
               Service: web
              Upstream: socat => :9191
       Public listener: Disabled

...
```

我们可以建立一个连接试一下：

```text
$ nc 127.0.0.1 9191
hello
hello
```

**此时代理之间的服务通信连接已经被加密和鉴权过了。**我们可以通过 TLS 连接和 “socat” 服务进行通信。而本地的连接则是未加密的，但是在生产环境，这是一个本地连接。所有出入机器的流量都会被加密。

### 注册依赖服务

我们已经在开发模式用  `consul connect proxy` 建立了连接。实际上，并不需要通过 Connect 来建立连接。通过在辅助选项里注册 “web” 服务和 “socat” 服务间的上游依赖就可以实现：

```text
$ cat <<EOF | sudo tee ./consul.d/web.json
{
  "service": {
    "name": "web",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "proxy": {
          "upstreams": [{
             "destination_name": "socat",
             "local_bind_port": 9191
          }]
        }
      }
    }
  }
}
EOF
```

这里注册为 “web” 服务注册了一个 sidecar 代理，它监听端口 9191，目标服务是 “socat”。这样，“web” 服务就可以通过本地端口来和 “socat” 进行通信，而不必直接与其连接。

好，用 `consul reload` 重新加载配置，再用 `Ctrl-C` 退掉之前还在运行的上游监听器（不要退错了，就是上面的那个 `-upstream socat:9191`）。现在我们可以用配置文件来启动 web 代理了：

```text
$ consul connect proxy -sidecar-for web
==> Consul Connect proxy starting...
    Configuration mode: Agent API
        Sidecar for ID: web
              Proxy ID: web-sidecar-proxy

==> Log data will now stream in as it occurs:

    2018/10/09 12:34:20 [INFO] 127.0.0.1:9191->service:default/socat starting on 127.0.0.1:9191
    2018/10/09 12:34:20 [INFO] Proxy loaded config and ready to serve
    2018/10/09 12:34:20 [INFO] TLS Identity: spiffe://df34ef6b-5971-ee61-0790-ca8622c3c287.consul/ns/default/dc/dc1/svc/web
    2018/10/09 12:34:20 [INFO] TLS Roots   : [Consul CA 7]
```

注意日志第一行，代理从本地 agent 发现并设置了一个 9191 端口的监听器，以如上面配置的那样代理 socat 服务。

你也可以看到这串身份 URL，它从 agent 加载认证信息，标识为 “web” 服务，并且加入信任的根 CA。

开个新连接验证一下吧：

```text
$ nc 127.0.0.1 9191
hello
hello
```

### Intention 的访问控制

Intention 用于定义服务通信。我们的连接之所以会成功是因为我们处在开发模式，ACL 默认处于 “allow all”（允许所有）状态。

让我们加一条从 web 到 socat 的禁止权限：

```text
$ consul intention create -deny web socat
Created: web => socat (deny)
```

再去连接就会失败：

```text
$ nc 127.0.0.1 9191

```

删除该 Intention 之后就又好了：

```text
$ consul intention delete web socat
Intention deleted.
$ nc 127.0.0.1 9191
hello
hello
```

Intention 允许服务通过 Consul 的中央控制台配置。参阅[文档](https://consul.io/docs/connect/intentions.html)获取更多信息。

注意，目前版本的 Consul，更改 Intention 不会影响到已经存在的连接。因此，你需要建立一个新的连接才能看到效果，未来的版本可能会改进。

### 探索 Connect

参阅 [Connect 文档](https://kingfree.gitbook.io/consul/connect)以获取更多关于 Connect 的信息，比如 Envoy 代理、Docker 和 Kubernetes。

