# 通过Consul Service Mesh连接服务

除了使用DNS接口或HTTP 接口直接提供服务的IP地址外，Consul还可以通过随每个服务实例在本地部署的Sidecar Proxy将服务彼此连接。 这种类型的部署（控制服务实例之间网络流量的本地代理）就是Service Mesh(服务网格)。 由于Sidecar Proxy可连接你的注册服务，因此在文档中有时将Consul的Service Mesh(服务网格)功能称为Consul Connect。 该术语是该功能的旧名称，在我们更新教程和文档时将不再使用该术语。
Consul Service Mesh(服务网格)使你可以在不修改服务代码的情况下保护和监控服务之间的通信。 Consul通过配置sidecar proxy以在你的服务之间建立mutual TLS（mTLS，双向TLS ），并根据其注册名称允许或拒绝它们之间的通信。 由于Sidecar Proxy控制着所有服务到服务的流量，因此它们可以收集有关它们的指标，并将其导出到Prometheus等第三方聚合器。

你还可以将应用程序与Consul Connect进行本地集成(https://www.consul.io/docs/connect/native.html)，以实现最佳性能和安全性。

使用Service Mesh(服务网格)注册服务与正常注册服务相似。 在本教程中，你将：

- 启动一个服务。
- 按常规注册它，但要附加一个connect字段。
- 再注册一个proxy以与服务进行通信。
- 启动Sidecar Proxy。
- 练习阻止与服务的连接。

#### 准备条件
##### 本地环境
本教程的唯一要求就是在本地环境中有安装Consul二进制文件。

> 安全警告：为简化起见，本教程以开发模式演示了Consul Connect Service Mesh(服务网格)功能，这不是在生产环境中推荐的部署Consul的安全方法。 请阅读Connect生产环境教程，以了解有关安全部署Consul Connect Service Mesh(服务网格)的信息。

#### 互动教程
我们还提供了涵盖本教程相同概念的交互式教程。 如果你不想在本地设置演示环境，请单击“显示教程”按钮以启动演示环境。

> 注意：交互式教程并未使用与本教程中列出的完全一致的指令。 你可以将交互式教程这个实验平台视为扩展本教程中介绍的概念的一种方法。

#### 启动一个对Consul无感知的服务

首先启动一个对Consul无感知的服务。 这里你将使用socat指令启动一个最基础的echo服务，该服务将在本教程中充当上游服务的角色。 在生产环境中，该服务可能是数据库，后端或其他服务依赖的任何服务。

Socat是一个具有数十年历史的Unix实用程序，它没有加密或TLS协议的概念。 你将使用它来证明Consul Connect Service Mesh(服务网格)为你解决了这些问题。 如果你的计算机上未安装socat，可以通过软件包管理器来安装使用。比如MacOS下可以：

	brew install socat

启动socat服务，并指定它侦听8181端口上的TCP连接：

	$ socat -v tcp-l:8181,fork exec:"/bin/cat"
	
	(base) c02ym00wjg5h:~ linyu$ socat -v tcp-l:8181,fork exec:"/bin/cat"
	> 2020/08/08 17:03:34.377599  length=6 from=0 to=5
	hello
	< 2020/08/08 17:03:34.378130  length=6 from=0 to=5
	hello
	> 2020/08/08 17:03:43.203669  length=12 from=6 to=17
	testing 123
	< 2020/08/08 17:03:43.204088  length=12 from=6 to=17
	testing 123

你可以直接通过`nc`（netcat）命令在对应的端口上连接到echo服务来验证它是否正常工作。 连接后，输入一些文本，然后按Enter。 你输入的文本将会回显在控制台：

	(base) C02YM00WJG5H:consul-learn linyu$ nc 127.0.0.1 8181
	hello
	hello
	testing 123
	testing 123

#### 注册该服务并通过Consul来代理它
接下来，就像在教程的前一个部分所做的那样，我们通过编写一个新的服务定义向Consul注册该服务。 这次，你的服务定义中将包含`connect`字段，该字段将在后端服务实例边上注册sidecar proxy， 用来处理此后端服务实例的流量。

添加一个叫做**socat.json**的文件到**consul.d**目录，文件中的内容如下（可以拷贝并粘贴除`$`以外的内容）:

	$ echo '{
	  "service": {
	    "name": "socat",
	    "port": 8181,
	    "connect": {
	      "sidecar_service": {}
	    }
	  }
	}' > ./consul.d/socat.json

现在运行`consul reload`或者发送一个SIGHUP信号给Consul从而使其读取新的服务定义文件。

可以看看刚刚添加的服务定义中的“`connect`”字段。 该空配置会通知Consul在动态分配的端口上为此进程注册一个Sidecar Proxy。 当你通过命令行启动时，它还会使用合理的默认值，Consul将使用该默认值来配置Proxy。 Consul不会自动为你启动 proxy 进程。 这是因为Consul Connect Service Mesh(服务网格)允许你选择要使用的proxy。

Consul原生拥有用于测试目的的L4 proxy，和对Envoy的一流支持——你应该将其用于生产环境部署，以及7层(L7)流量管理。 在本教程中，你将使用L4 proxy，因为与Envoy不同的是，它是Consul自带的，不需要任何额外的安装。

使用`consul connect proxy`命令在另一个终端窗口中启动proxy进程，并指定其对应的服务实例。

	$ consul connect proxy -sidecar-for socat
	
	(base) c02ym00wjg5h:~ linyu$ consul connect proxy -sidecar-for socat
	==> Consul Connect proxy starting...
	    Configuration mode: Agent API
	        Sidecar for ID: socat
	              Proxy ID: socat-sidecar-proxy
	
	==> Log data will now stream in as it occurs:
	
	    2020-08-08T17:08:19.089+0800 [INFO]  proxy: Proxy loaded config and ready to serve
	    2020-08-08T17:08:19.089+0800 [INFO]  proxy: Parsed TLS identity: uri=spiffe://f85d5c94-24e5-c805-3681-b87674570927.consul/ns/default/dc/dc1/svc/socat roots=[pri-12c9ah6t.consul.ca.f85d5c94.consul]
	    2020-08-08T17:08:19.089+0800 [INFO]  proxy: Starting listener: listener="public listener" bind_addr=0.0.0.0:21000


#### 注册一个依赖服务和proxy
接下来，注册一个称为“`web`”的下游服务。 类似于`socat`服务定义，用于web服务的配置文件将包含一个指定sidecar的`connect`字段，但是与socat定义不同的是，此处的配置不是空的。 相反，它指定`web`对`socat`服务的上游依赖性，以及proxy将侦听的端口。

	$ echo '{
	  "service": {
	    "name": "web",
	    "connect": {
	      "sidecar_service": {
	        "proxy": {
	          "upstreams": [
	            {
	              "destination_name": "socat",
	              "local_bind_port": 9191
	            }
	          ]
	        }
	      }
	    }
	  }
	}' > ./consul.d/web.json

同样，使用`consul reload` 或 SIGHUP 来重启Consul。这会为`web` 服务注册一个sidebar proxy。 该proxy将监听9191端口并建立到`socat`的mTLS(双向 TLS, **m**utual **T**ransport **L**ayer **S**ecurity)链接。

>**mTLS**(mutual Transport Layer Security)
>
>所有的请求和回复，都会进行双向TLS认证。在加密的情形下，请求与回复的内容无法被网络嗅探。

如果我们运行的是真正的Web服务，它将在回环地址上与其proxy进行通信。 proxy将对其流量进行加密，并通过网络将其发送到socat服务的sidecar proxy。 `socat`的sidecar proxy将解密流量并将其通过本地8181端口上的回环地址发送到`socat`服务。由于没有Web服务在运行，因此你将通过在我们指定的（9191）端口上与它的proxy进行通信来假装为真实的Web服务 。

在启动proxy进程之前，请确认你无法在9191端口上连接到`socat`服务。如果运行以下命令应立即退出，因为在9191端口上`socat`没有侦听任何内容（`socat`在8181上侦听）。

	$ nc 127.0.0.1 9191

现在使用sidecar注册的配置文件启动proxy

	$ consul connect proxy -sidecar-for web
	
	==> Consul Connect proxy starting...
	    Configuration mode: Agent API
	        Sidecar for ID: web
	              Proxy ID: web-sidecar-proxy
	
	==> Log data will now stream in as it occurs:
	
	    2019/07/24 13:32:10 [INFO] 127.0.0.1:9191->service:default/socat starting on 127.0.0.1:9191
	    2019/07/24 13:32:10 [INFO] Proxy loaded config and ready to serve
	    2019/07/24 13:32:10 [INFO] TLS Identity: spiffe://287133f6-3d1e-8fb0-a0c5-fb9d5a95d53c.consul/ns/default/dc/dc1/svc/web
	    2019/07/24 13:32:10 [INFO] TLS Roots   : [Consul CA 7]
	    2019/07/24 13:32:10 [INFO] public listener starting on 0.0.0.0:21001



>注意，在第一行日志中，proxy在端口9191上设置了本地侦听器，其会将请求代理到`socat`服务，就像我们在sidecar注册中配置的那样。 随后的日志行列出了从proxy加载的证书的身份URL（将其标识为“ `web`”服务）以及代理知晓的一组受信任的根证书。

尝试再次在端口9191上连接至`socat`。这一次它应该可以工作并回显你发送的文本。

	$ nc 127.0.0.1 9191
	
	hello
	hello

可以使用Crl+c关闭链接。

web proxy和socat proxy之间的通信通过mTLS连接进行加密和授权，而每个服务与其Sidecar proxy之间的通信未加密。 在生产环境中，服务应当仅接受回环连接。 机器进出的任何流量都应通过proxy来完成，因此始连接终会被加密。

>安全警告：Consul Connect Service Mesh安全模型在使用proxy时需要信任回环连接。 你可以使用网络命名空间之类的工具进一步保障回环连接的安全性。

#### 使用intentions指令来控制通信
`intention`定义允许哪些服务与其他服务进行通信。 上面的连接可以成功是因为在开发模式下，ACL系统默认情况下（以及默认的intention策略）为“全部允许”。

创建`intention`用于阻断从`web`服务到`socat`服务的访问，该指令指定了`intention`的策略、源服务和目标服务。

	$ consul intention create -deny web socat
	
	Created: web => socat (deny)

现在，再次尝试连接，将会出现失败：

	$ nc 127.0.0.1 9191

#### 删除intention：

	$ consul intention delete web socat
	Intention deleted.

再次尝试连接将会成功

	$ nc 127.0.0.1 9191
	
	hello
	hello

`intention`使你可以像传统防火墙一样对网络进行分隔，但是它们依赖于服务的逻辑名称（例如“ `web`”或“`socat`”），而不是每个服务实例的IP地址。 在文档中了解有关`intention` （https://www.consul.io/docs/connect/intentions.html）的更多信息。

> 注意：更改intention不会影响与Consul当前的现有连接。 你必须建立新的连接才能查看intention改变的影响。

#### 下一步
在本教程中，你在单个agent上配置了服务，并使用Consul Connect Service Mesh进行自动连接授权和加密。 想了解Consul Connect Service Mesh的其他功能，可查看Getting Started with Consul Service Mesh（https://learn.hashicorp.com/consul?track=gs-consul-service-mesh#gs-consul-service-mesh）。

接下来，将带你按照《**在Consul KV中存储数据**》的教程，探索如何使用Consul的键值（KV）进行服务配置存储。
