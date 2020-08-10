# 在Consul KV中存储数据

除了提供服务发现、集成健康检查以及安全的网络链路外，Consul还包含kv存储，你可以将其用于动态地配置应用程序，协调服务，管理leader选举或充当Vault的数据后端，以及数不胜数的其他用途。

在本教程中，你将使用命令行探索Consul键值存储（Consul KV）。 本教程假定“启动Consul Agent”教程中的Consul agent仍在运行。 如果没有，你可以通过运行`consoul proxy -dev`启动一个新的agent。 Consul KV在Consul agent上自动开启； 你无需在Consul的配置中启用它。

有2种方式可以与Consul KV存储进行交互：HTTP API和命令行（CLI）。 在本教程中，你将使用命令行（CLI）。 请参阅HTTP API文档（https://www.consul.io/api/kv.html），以了解服务如何通过HTTP接口与Consul KV进行交互。

#### 添加数据
首先，使用consul kv的 `put`命令将一些value存入 KV存储中。 `put`命令后面的第一个参数是key，第二个参数是value。

	$ consul kv put redis/config/minconns 1
	Success! Data written to: redis/config/minconns

在上面这里，key 是 redis/config/maxconns ，value 是 25

	$ consul kv put redis/config/maxconns 25
	Success! Data written to: redis/config/maxconns

请注意，使用下面输入的key（“`redis/config/users/admin`”），你将设置一个值为`42`的flag。key支持设置64位整数flag, 该值不是在Consul内部使用，但是客户端可以使用该值将元数据添加到任何KV对中。

	$ consul kv put -flags=42 redis/config/users/admin abcd1234
	Success! Data written to: redis/config/users/admin

#### 查询数据
现在，查询你刚输入的任何一个key

	$ consul kv get redis/config/minconns
	1

Consul保留了一些有关键值对的其他元数据。 使用-detailed命令行标志可以检索一些元数据（包括你设置的flag）。

	$ consul kv get -detailed redis/config/users/admin
	
	CreateIndex      14
	Flags            42
	Key              redis/config/users/admin
	LockIndex        0
	ModifyIndex      14
	Session          -
	Value            abcd1234

可使用`recurse`选项列出存储中的所有key。 其结果将按字典顺序返回。

	$ consul kv get -recurse
	
	redis/config/maxconns:25
	redis/config/minconns:1
	redis/config/users/admin:abcd1234

#### 删除数据

使用`delete`指令从Consul KV存储中删除数据：

	$ consul kv delete redis/config/minconns
	Success! Deleted key: redis/config/minconns

Consul允许你以类似于文件夹的方式与key进行交互。 尽管KV存储中的所有key实际上都是扁平化存储的，但Consul允许你将具有相同前缀的key作为一组进行操作，就像它们在文件夹或子文件夹中。

可以使用`recurse`指令删除所有带`redis`前缀的key

	$ consul kv delete -recurse redis
	Success! Deleted keys with prefix: redis

#### 修改数据
更新某个key的value

	$ consul kv put foo bar
	Success! Data written to: foo

查询更新后的key

	$ consul kv get foo
	bar

#### 下一步
在本教程中，你在Consul的KV存储中添加，查看，修改和删除了一些键值对。

Consul可以使用Check-And-Set（CAS）操作执行原子键更新，并且包含写入键和值的其他复杂选项。 你可以在consul kv put命令的帮助页面上浏览这些选项。

	$ consul kv put -h

这些只是API支持的几个示例。 有关完整的文档，请参阅HTTP API文档（https://www.consul.io/api/kv.html）和CLI文档(https://www.consul.io/docs/commands/kv.html)。

现在你已经知道了Consul包含一些有趣的关于服务注册，键，值和intention的数据，请继续浏览**使用Consul Web UI**教程以在Consul Web UI中探索所有这些数据。
