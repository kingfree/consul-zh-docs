# 使用Consul Web UI

Consul的Web UI允许你通过图形用户界面查看Consul并与之交互，这可以降低新用户的进入门槛，并简化故障排除。

如果你在生产环境中运行Consul，则需要在Consul的配置文件中启用UI或使用`-ui`命令行标志，但是由于这里你的agent程序在开发模式下运行，因此会自动启用UI。


#### 导航到UI界面
如果你已经把前面教程中启动的consul节点关掉了，你也可以访问Consul Web UI 的这个在线demo来探索本节中的内容。

否则，如果你没有关掉前面启动的consul节点，你则可以更加真切的按照本教程走下去。现在打开你的浏览器，输入 **http://localhost:8500/ui** 即可以打开Cosnul Web UI 界面。

你将会看到一个顶部是粉色菜单的页面。
#### 查看服务

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNKnASnnTNp8Dc4tvtRoMfdL0YicDjwmmmARAbiaLvFqSSGiatJj3acPHZw/640?wx_fmt=png)

>注：最新版的已经不长上面那样了，而是下面这样👇

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNhsKerticHMmhCGB8E6atbPdZCyBiaRGnvrEogUOB4q0AibDE6QoNlq0Dw/640?wx_fmt=png)

首先进入的落地页是Service页面，该页面将所有注册的服务列了出来，包括他们的服务名，健康状况，tags，类型以及资源。你可以点击某个服务来查看有关其实例个数、每个实例健康状况以及该实例注册到哪个agent了等相关的更多信息。

你可以根据服务的名称，标签，状态或其他搜索条件过滤页面上可见的服务。

> 尝试一下：在搜索栏中输入socat并按Enter键，以筛选出socat服务。

你可以通过单击来了解各个服务的详细信息。

> 尝试一下：单击web、服务以浏览其相关信息。 现在，选择一个列出的web服务实例，以查看实例-实例的信息。

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNWI4kf6MVopicouiax5ddSFae1cetpiaibaqchgGsHyzF07GCHAkwYVTXcA/640?wx_fmt=png)

单击instance实例：

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNYvlrCfu3wibxZJMo68zEaVVQT5rJmd8GzCsGY64ypVJSOh3Hic96FTtA/640?wx_fmt=png)

Proxy Infos

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNMNXojicltwv7xLTmEShicJRo1x8dkCOrPMBQp4re5LhN4VCwGXNJaXDA/640?wx_fmt=png)


#### 查看节点
接下来，点击粉红色的顶部导航栏中“Nodes”菜单，跳转到节点页面。 你将在其中找到整个数据中心的概述，包括每个节点的运行状况。 你可以选择各个节点以了解其运行状况检查，注册的服务，往返时间和锁定会话。

你还可以按健康状态过滤节点，或在搜索栏中搜索它们。

> 尝试一下：从顶部菜单栏中选择节点页面，然后单击本地计算机这个节点。

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNDoqpj4RgXAzdx4QfcjDgnY0ae0gnNDYpsFbTCSY3TlfGQZgmpOJXaA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNFpadD7f7Y2vAo7BC2f5UmnHVpjEsliamIMib2Comjx5tbk99QRWTtPYw/640?wx_fmt=png)


![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNpiatQDnWoxFffFefLorpFryuicicCPFThAl9B2T3MMpMDF3zDUVeFrzIQ/640?wx_fmt=png)


#### 管理KV存储
在顶部导航中，单击“Key/Value”菜单以查看Consul KV的页面。 如果你使用的是与先前教程相同的代理，则应该看到一个key `foo`。

![](https://mmbiz.qlogo.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNKjSicrErkCs2gm0LkibEIDOaS0Fb5ZCeDx8yP6aSSwOChO4aOHk9pPkg/0?wx_fmt=png)

key页面具有类似文件夹的结构。根据其键前缀出现嵌套。 例如，你可以为每个应用程序，业务功能或两者的嵌套组合使用一个文件夹。

> 尝试一下：在主页上，单击蓝色的“Create”按钮以添加新的键值对。 用`Alice`作为key `redis/user`的value。 现在，使用key `redis/password` 和value `123`创建另一个键值对。在主页上，请注意只有一个新条目，称为“redis”，旁边是文件夹图标。

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNvrum9vJonJzNibSia6hJ1UPQ00vTUvvOSMeJLslswiagtlibETefDrSXLA/640?wx_fmt=png)

当你单击文件夹时，Consul将自动在该文件夹下嵌套新key，而无需键入前缀。

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNhfaGOTnQnyPZicTc8hPXL91Uk81IDfsYia7O5W0BGtqsXGAeZ1U2glhA/640?wx_fmt=png)

#### 管理访问控制列表

Consul使用访问控制列表（ACL）来保护Web UI，API，CLI，服务和sidecar proxy之间通信。 你需要在Consul数据中心中配置ACL以保护它在生产环境中的安全，但是，在开发模式下的代理上没有启用它们，因此目前在“ACL”页面上没有多少内容。

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNQZS7HAxlr7Uh68431LKvBB6zXGlqviaRmRAiayibbvB53rAlCETylWHkw/640?wx_fmt=png)


通过限制UI中各个页面的读取，写入和更新权限，可以使用ACL保护UI本身。 为此，你可以创建具有适当权限的令牌，然后将其添加到ACL页面下的UI中。 要删除访问权限，只需从令牌列表中的令牌操作菜单中选择“停止使用”。


#### 管理intention
单击“intention”菜单项以导航到UI中的intention页面。 目前还没有任何intention，但是如果你仍在运行与先前教程相同的agent，则可以通过创建intention来阻止`web`服务与`socat`服务之间的通信。

比如，如果我们按之前的教程，创建一个阻止web到socat的连接：

	(base) c02ym00wjg5h:consul-learn linyu$ consul intention create -deny web socat

那么在intentions菜单下，你将会看到：

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNsZndV1jhrW2MyDdzRpYzlp6DNY8h5fycOYH2TyjibgUv11ZLicV9J7HA/640?wx_fmt=png)


而删除该intention之后，UI界面上也将看不到了：

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNRa6pyItV1eic6D9b8EgptYZ0Q2vyBV0fBmGv48ib2oFpV5j8HtcNfbaA/640?wx_fmt=png)

当然，你也可以使用UI提供的创建、删除、编辑功能完成同样的效果。

#### 调整UI配置

单击菜单栏最右侧的“Settings”。 你可以在此处编辑用户界面的设置。

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNeWYXSc354TLwy0kXf18Ks2bWWSxM0o4ouhtG7PaicCwmR8RNSWUzZTg/640?wx_fmt=png)

如果已设置指标仪表板来监视服务，则可以在设置页面上添加一个链接，该链接将自动为该服务名称和数据中心填充占位符，并从其UI页面链接到每个服务的指标。

你还可以选择是否要设置使用阻塞式的查询来实时更新UI，而不是刷新时才更新。 默认情况下启用了此功能，你也可以将其禁用，因为它可能会影响页面性能。

#### UI任务表

你可能已经注意到，Web UI的某些页面是只读的，而其他页面是可交互的。 下表是每个页面的CRUD可操作性的列表。

![](https://mmbiz.qpic.cn/mmbiz_png/XsgEbl9EdmnNjHTMIhVzgVnA38r1y9ibNDgkNv4ZmSOqpoLpZwxqd8oF3S7tHHA6RMoOkmzZGbjjbo2EbgJxd3g/640?wx_fmt=png)

#### 下一步
现在你可以轻松查看和使用Web UI了，请尝试使用Consul 命令行工具完成此处列出的相同任务。

到目前为止，你已经探索了Consul的核心功能，包括服务发现，使用Service Mesh保护服务以及使用KV存储。 继续下一个教程“**创建本地Consul数据中心**”，以了解如何通过将多个Consul agent连接在一起来设置Consul数据中心。

>注意：下一个教程依赖VirtualBox和Vagrant一次在你的计算机上运行多个Consul agent。 如果你还没有下载它们，那么现在下载是个不错的选择。
