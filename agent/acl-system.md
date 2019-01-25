# ACL System

Consul provides an optional Access Control List \(ACL\) system which can be used to control access to data and APIs. The ACL is [Capability-based](https://en.wikipedia.org/wiki/Capability-based_security), relying on tokens which are associated with policies to determine which fine grained rules can be applied. Consul's capability based ACL system is very similar to the design of [AWS IAM](https://aws.amazon.com/iam/).

### ACL System Overview <a id="acl-system-overview"></a>

The ACL system is designed to be easy to use and fast to enforce while providing administrative insight. At the highest level, there are two major components to the ACL system:

* **ACL Policies** - Policies allow the grouping of a set of rules into a logical unit that can be reused and linked with many tokens.
* **ACL Tokens** - Requests to Consul are authorized by using bearer token. Each ACL token has a public Accessor ID which is used to name a token, and a Secret ID which is used as the bearer token used to make requests to Consul.

ACL tokens and policies are managed by Consul operators via Consul's [ACL API](https://www.consul.io/api/acl/acl.html), [ACL CLI](https://www.consul.io/docs/commands/acl.html), or systems like [HashiCorp's Vault](https://www.vaultproject.io/docs/secrets/consul/index.html).

#### ACL Policies <a id="acl-policies"></a>

An ACL policy is a named set of rules and is composed of the following elements:

* **ID** - The policies auto-generated public identifier.
* **Name** - A unique meaningful name for the policy.
* **Rules** - Set of rules granting or denying permissions. See the [Rule Specification](https://www.consul.io/docs/agent/acl-rules.html#rule-specification) documentation for more details.
* **Datacenters** - A list of datacenters the policy is valid within.

**Builtin Policies**

* **Global Management** - Grants unrestricted privileges to any token that uses it. When created it will be named `global-management` and will be assigned the reserved ID of `00000000-0000-0000-0000-000000000001`. This policy can be renamed but modification of anything else including the rule set and datacenter scoping will be prevented by Consul.

#### ACL Tokens <a id="acl-tokens"></a>

ACL tokens are used to determine if the caller is authorized to perform an action. An ACL token is composed of the following elements:

* **Accessor ID** - The token's public identifier.
* **Secret ID** -The bearer token used when making requests to Consul.
* **Description** - A human readable description of the token. \(Optional\)
* **Policy Set** - The list of policies that are applicable for the token.
* **Locality** - Indicates whether the token should be local to the datacenter it was created within or created in the primary datacenter and globally replicated.

**Builtin Tokens**

During cluster bootstrapping when ACLs are enabled both the special `anonymous` and the `master` token will be injected.

* **Anonymous Token** - The anonymous token is used when a request is made to Consul without specifying a bearer token. The anonymous token's description and policies may be updated but Consul will prevent this tokens deletion. When created, it will be assigned `00000000-0000-0000-0000-000000000002` for its Accessor ID and `anonymous` for its Secret ID.
* **Master Token** - When a master token is present within the Consul configuration, it is created and will be linked With the builtin Global Management policy giving it unrestricted privileges. The master token is created with the Secret ID set to the value of the configuration entry.

**Authorization**

The token Secret ID is passed along with each RPC request to the servers. Consul's [HTTP endpoints](https://www.consul.io/api/index.html) can accept tokens via the `token` query string parameter, the `X-Consul-Token` request header, or an [RFC6750](https://tools.ietf.org/html/rfc6750) authorization bearer token. Consul's [CLI commands](https://www.consul.io/docs/commands/index.html) can accept tokens via the `token` argument, or the `CONSUL_HTTP_TOKEN` environment variable.

If no token is provided for an HTTP request then Consul will use the default ACL token if it has been configured. If no default ACL token was configured then the anonymous token will be used.

**ACL Rules and Scope**

The rules from all policies linked with a token are combined to form that token's effective rule set. Policy rules can be defined in either a whitelist or blacklist mode depending on the configuration of [`acl_default_policy`](https://www.consul.io/docs/agent/options.html#acl_default_policy). If the default policy is to "deny" access to all resources, then policy rules can be set to whitelist access to specific resources. Conversely, if the default policy is “allow” then policy rules can be used to explicitly deny access to resources.

The following table summarizes the ACL resources that are available for constructing rules:

| Resource | Scope |
| :--- | :--- |
| [`acl`](https://www.consul.io/docs/agent/acl-system.html#acl-rules) | Operations for managing the ACL system [ACL API](https://www.consul.io/api/acl/acl.html) |
| [`agent`](https://www.consul.io/docs/agent/acl-system.html#agent-rules) | Utility operations in the [Agent API](https://www.consul.io/api/agent.html), other than service and check registration |
| [`event`](https://www.consul.io/docs/agent/acl-system.html#event-rules) | Listing and firing events in the [Event API](https://www.consul.io/api/event.html) |
| [`key`](https://www.consul.io/docs/agent/acl-system.html#key-value-rules) | Key/value store operations in the [KV Store API](https://www.consul.io/api/kv.html) |
| [`keyring`](https://www.consul.io/docs/agent/acl-system.html#keyring-rules) | Keyring operations in the [Keyring API](https://www.consul.io/api/operator/keyring.html) |
| [`node`](https://www.consul.io/docs/agent/acl-system.html#node-rules) | Node-level catalog operations in the [Catalog API](https://www.consul.io/api/catalog.html), [Health API](https://www.consul.io/api/health.html), [Prepared Query API](https://www.consul.io/api/query.html), [Network Coordinate API](https://www.consul.io/api/coordinate.html), and [Agent API](https://www.consul.io/api/agent.html) |
| [`operator`](https://www.consul.io/docs/agent/acl-system.html#operator-rules) | Cluster-level operations in the [Operator API](https://www.consul.io/api/operator.html), other than the [Keyring API](https://www.consul.io/api/operator/keyring.html) |
| [`query`](https://www.consul.io/docs/agent/acl-system.html#prepared-query-rules) | Prepared query operations in the [Prepared Query API](https://www.consul.io/api/query.html) |
| [`service`](https://www.consul.io/docs/agent/acl-system.html#service-rules) | Service-level catalog operations in the [Catalog API](https://www.consul.io/api/catalog.html), [Health API](https://www.consul.io/api/health.html), [Prepared Query API](https://www.consul.io/api/query.html), and [Agent API](https://www.consul.io/api/agent.html) |
| [`session`](https://www.consul.io/docs/agent/acl-system.html#session-rules) | Session operations in the [Session API](https://www.consul.io/api/session.html) |

Since Consul snapshots actually contain ACL tokens, the [Snapshot API](https://www.consul.io/api/snapshot.html) requires a token with "write" privileges for the ACL system.

The following resources are not covered by ACL policies:

1. The [Status API](https://www.consul.io/api/status.html) is used by servers when bootstrapping and exposes basic IP and port information about the servers, and does not allow modification of any state.
2. The datacenter listing operation of the [Catalog API](https://www.consul.io/api/catalog.html#list-datacenters) similarly exposes the names of known Consul datacenters, and does not allow modification of any state.
3. The [connect CA roots endpoint](https://www.consul.io/api/connect/ca.html#list-ca-root-certificates) exposes just the public TLS certificate which other systems can use to verify the TLS connection with Consul.

Constructing rules from these policies is covered in detail in the [Rule Specification](https://www.consul.io/docs/agent/acl-system.html#rule-specification) section below.

### [»](https://www.consul.io/docs/agent/acl-system.html#configuring-acls)Configuring ACLs <a id="configuring-acls"></a>

ACLs are configured using several different configuration options. These are marked as to whether they are set on servers, clients, or both.

| Configuration Option | Servers | Clients | Purpose |
| :--- | :--- | :--- | :--- |
| [`acl.enabled`](https://www.consul.io/docs/agent/options.html#acl_enabled) | `REQUIRED` | `REQUIRED` | Controls whether ACLs are enabled |
| [`acl.default_policy`](https://www.consul.io/docs/agent/options.html#acl_default_policy) | `OPTIONAL` | `N/A` | Determines whitelist or blacklist mode |
| [`acl.down_policy`](https://www.consul.io/docs/agent/options.html#acl_down_policy) | `OPTIONAL` | `OPTIONAL` | Determines what to do when the remote token or policy resolution fails |
| [`acl.policy_ttl`](https://www.consul.io/docs/agent/options.html#acl_policy_ttl) | `OPTIONAL` | `OPTIONAL` | Determines time-to-live for cached ACL Policies |
| [`acl.token_ttl`](https://www.consul.io/docs/agent/options.html#acl_token_ttl) | `OPTIONAL` | `OPTIONAL` | Determines time-to-live for cached ACL Tokens |

A number of special tokens can also be configured which allow for bootstrapping the ACL system, or accessing Consul in special situations:

| Special Token | Servers | Clients | Purpose |
| :--- | :--- | :--- | :--- |
| [`acl.tokens.agent_master`](https://www.consul.io/docs/agent/options.html#acl_tokens_agent_master) | `OPTIONAL` | `OPTIONAL` | Special token that can be used to access [Agent API](https://www.consul.io/api/agent.html) when remote bearer token resolution fails; used for setting up the cluster such as doing initial join operations, see the [ACL Agent Master Token](https://www.consul.io/docs/agent/acl-system.html#acl-agent-master-token)section for more details |
| [`acl.tokens.agent`](https://www.consul.io/docs/agent/options.html#acl_tokens_agent) | `OPTIONAL` | `OPTIONAL` | Special token that is used for an agent's internal operations, see the [ACL Agent Token](https://www.consul.io/docs/agent/acl-system.html#acl-agent-token) section for more details |
| [`acl.tokens.master`](https://www.consul.io/docs/agent/options.html#acl_tokens_master) | `OPTIONAL` | `N/A` | Special token used to bootstrap the ACL system, see the [Bootstrapping ACLs](https://www.consul.io/docs/agent/acl-system.html#bootstrapping-acls) section for more details |
| [`acl.tokens.default`](https://www.consul.io/docs/agent/options.html#acl_tokens_default) | `OPTIONAL` | `OPTIONAL` | Default token to use for client requests where no token is supplied; this is often configured with read-only access to services to enable DNS service discovery on agents |

All of these tokens except the `master` token can all be introduced or updated via the [/v1/agent/token API](https://www.consul.io/api/agent.html#update-acl-tokens).

**ACL Agent Master Token**

Since the [`acl.tokens.agent_master`](https://www.consul.io/docs/agent/options.html#acl_tokens_agent_master) is designed to be used when the Consul servers are not available, its policy is managed locally on the agent and does not need to have a token defined on the Consul servers via the ACL API. Once set, it implicitly has the following policy associated with it

```text
agent "<node name of agent>" {
  policy = "write"
}
node_prefix "" {
  policy = "read"
}
```

**ACL Agent Token**

The [`acl.tokens.agent`](https://www.consul.io/docs/agent/options.html#acl_tokens_agent) is a special token that is used for an agent's internal operations. It isn't used directly for any user-initiated operations like the [`acl.tokens.default`](https://www.consul.io/docs/agent/options.html#acl_tokens_default), though if the `acl.tokens.agent_token` isn't configured the `acl.tokens.default` will be used. The ACL agent token is used for the following operations by the agent:

1. Updating the agent's node entry using the [Catalog API](https://www.consul.io/api/catalog.html), including updating its node metadata, tagged addresses, and network coordinates
2. Performing [anti-entropy](https://www.consul.io/docs/internals/anti-entropy.html) syncing, in particular reading the node metadata and services registered with the catalog
3. Reading and writing the special `_rexec` section of the KV store when executing [`consul exec`](https://www.consul.io/docs/commands/exec.html) commands

Here's an example policy sufficient to accomplish the above for a node called `mynode`:

```text
node "mynode" {
  policy = "write"
}
service_prefix "" {
  policy = "read"
}
key_prefix "_rexec" {
  policy = "write"
}
```

The `service_prefix` policy needs read access for any services that can be registered on the agent. If [remote exec is disabled](https://www.consul.io/docs/agent/options.html#disable_remote_exec), the default, then the `key_prefix` policy can be omitted.

### Next Steps <a id="next-steps"></a>

Setup ACLs with the [Boostrapping guide](https://www.consul.io/docs/guides/acl.html) or continue reading about [ACL rules](https://www.consul.io/docs/agent/acl-rules.html).

