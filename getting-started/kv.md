# 7. KV 数据

作为对于提供服务发现和集成健康检查的补充，Consul 提供了一个简单的键值存储\(KV store\)。它可以用于保存动态配置、辅助服务间协作、建立领导选举以及其他开发者想要实现的功能。

本节要求你至少运行一个 Consul agent。

### 简单用法

为了演示它的易用性，我们使用若干个键。有两种方法可以访问 Consul KV 存储：通过 HTTP API 或者 Consul KV CLI。下面展示了通过 Consul KV CLI 的使用方法。进阶集成方案参见 [Consul KV HTTP API](https://www.consul.io/api/kv.html)。

首先来看看 KV 存储的样子，来问问 Consul 这个键的值吧：

```text
$ consul kv get redis/config/minconns
Error! No key exists at: redis/config/minconns
```

可见，由于 KV 存储里什么都没有，这个查询也没有结果。我们可以插入一些键值对进来：

```text
$ consul kv put redis/config/minconns 1
Success! Data written to: redis/config/minconns

$ consul kv put redis/config/maxconns 25
Success! Data written to: redis/config/maxconns

$ consul kv put -flags=42 redis/config/users/admin abcd1234
Success! Data written to: redis/config/users/admin
```

现在我们有了一些键，来查查看：

```text
$ consul kv get redis/config/minconns
1
```

用 `-detailed` 标识可以获取 Consul 保存的详细元数据：

```text
$ consul kv get -detailed redis/config/minconns
CreateIndex      207
Flags            0
Key              redis/config/minconns
LockIndex        0
ModifyIndex      207
Session          -
Value            1
```

对于 `redis/config/users/admin` 这个键我们设置了 `flag` 为 42。所有键都支持一个 64 位整数的标志位，它不用于 Consul 内部处理，但可以被客户端读取并赋予新的涵义。

通过 `-recurse` 选项可以递归查询子键，结果按字典序排列：

```text
$ consul kv get -recurse
redis/config/maxconns:25
redis/config/minconns:1
redis/config/users/admin:abcd1234
```

用 `delete` 可以删除一个键值对：

```text
$ consul kv delete redis/config/minconns
Success! Deleted key: redis/config/minconns
```

也可以用 `-recurse` 选项递归删除子键：

```text
$ consul kv delete -recurse redis
Success! Deleted keys with prefix: redis
```

要更新一个值，用 `put` 即可：

```text
$ consul kv put foo bar

$ consul kv get foo
bar

$ consul kv put foo zip

$ consul kv get foo
zip
```

Consul 可以原子性地进行检查并设置\(Check-And-Set\)操作，用 `-cas` 标识：

```text
$ consul kv put -cas -modify-index=123 foo bar
Success! Data written to: foo

$ consul kv put -cas -modify-index=123 foo bar
Error! Did not write to foo: CAS failed
```

如上，第一个 CAS 更新成功了，因为 index 是 123。第二个则失败了，因为此时的 index 不再是 123 了。

### 进阶用法

 参见 [Consul KV HTTP API](https://www.consul.io/api/kv.html) 或 [Consul KV CLI](https://www.consul.io/docs/commands/kv.html) 文档。



