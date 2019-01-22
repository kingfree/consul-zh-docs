# 1. 安装 Consul

请先安装 Consul 到你的机器上，Consul 为支持的平台和架构都发布了[二进制包](https://www.consul.io/downloads.html)。读者可以从[文档](https://www.consul.io/docs/install/index.html#compiling-from-source)中获取如何从源代码编译 Consul 的步骤。

### 安装 Consul

请下载适合你[操作系统的包](https://www.consul.io/downloads.html)，它是一个 zip 压缩文件。

下载完成后，unzip 解压缩。Consul 只有一个可执行文件 `consul`，任何包里的文件都可以删除而不会影响 Consul 的功能。

最后请检查 `consul` 是否在环境变量 `PATH` 下。Linux 和 Mac 设置步骤见[该页](https://stackoverflow.com/questions/14637979/how-to-permanently-set-path-on-linux)，Windows 见[该页](https://stackoverflow.com/questions/1618280/where-can-i-set-path-to-make-exe-on-windows)。

### 验证安装

安装完 Consul 后，可以在终端里键入 `consul` 检查是否可用。你将会看到这样的输出：

```text
$ consul
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    acl            Interact with Consul's ACLs
    agent          Runs a Consul agent
    ...
```

如果你看到找不到 `consul` 的错误提示，请检查环境变量 `PATH` 设置是否正确，以及 Consul 是否正确安装。

