# 5. 启用 ACL 系统

Consul 使用权限控制列表\(Access Control List, ACL\)实现 UI、API、CLI、服务和 agent 交流的安全性。交流和 RPC 的详细内容参见[该教程](https://kingfree.gitbook.io/consul/day-1-operations/agent-encryption)。要为你的集群实现安全性请先配置 ACL。

ACL 的核心思想是，通过将规则归类为策略\(policy\)，然后通过 token 来分配一条或多条策略。

本节要求 Consul 1.4 以上版本，建议阅读 [ACL 系统文档](https://www.consul.io/docs/agent/acl-system.html)。老版本请看[老的 ACL 文档](https://www.consul.io/docs/guides/acl-legacy.html)。

启用 ACL 系统需要多个步骤，我们将介绍几个必要的步骤。

1. 为所有 server 启用 ACL
2. 创建初始化 token
3. 创建 agent 策略
4. 创建 agent token
5. 将新 token 应用到各 server
6. 为 client 启用 ACL 并应用 agent token

文末还有一些辅助和可选的步骤。

### 第1步：为所有 Consul Server 启用 ACL

### 第2步：创建启动 Token

### 第3步：创建 Agent Token 策略

### 第4步：创建 Agent Token

### 第5步：将 Agent Token 加到其他 Server

### 第6步：为所有 Consul Client 启用 ACL

### 可选的 ACL 配置

### 总结

