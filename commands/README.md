# 命令行 \(CLI\)

Consul is controlled via a very easy to use command-line interface \(CLI\). Consul is only a single command-line application: `consul`. This application then takes a subcommand such as "agent" or "members". The complete list of subcommands is in the navigation to the left.

The `consul` CLI is a well-behaved command line application. In erroneous cases, a non-zero exit status will be returned. It also responds to `-h` and `--help` as you'd most likely expect. And some commands that expect input accept "-" as a parameter to tell Consul to read the input from stdin.

To view a list of the available commands at any time, just run `consul` with no arguments:

```text
$ consul
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    catalog        Interact with the catalog
    connect        Interact with Consul Connect
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators.
    intention      Interact with Connect service intentions
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    services       Interact with services
    snapshot       Saves, restores and inspects snapshots of Consul server state
    validate       Validate config files/directories
    version        Prints the Consul version
    watch          Watch for changes in Consul
```

To get help for any specific command, pass the `-h` flag to the relevant subcommand. For example, to see help about the `join` subcommand:

```text
$ consul join -h
Usage: consul join [options] address ...

  Tells a running Consul agent (with "consul agent") to join the cluster
  by specifying at least one existing member.

HTTP API Options

  -http-addr=<address>
     The `address` and port of the Consul HTTP agent. The value can be
     an IP address or DNS address, but it must also include the port.
     This can also be specified via the CONSUL_HTTP_ADDR environment
     variable. The default value is http://127.0.0.1:8500. The scheme
     can also be set to HTTPS by setting the environment variable
     CONSUL_HTTP_SSL=true.

  -token=<value>
     ACL token to use in the request. This can also be specified via the
     CONSUL_HTTP_TOKEN environment variable. If unspecified, the query
     will default to the token of the Consul agent at the HTTP address.

Command Options

  -wan
     Joins a server to another server in the WAN pool.
```

### Autocompletion <a id="autocompletion"></a>

The `consul` command features opt-in subcommand autocompletion that you can enable for your shell with `consul -autocomplete-install`. After doing so, you can invoke a new shell and use the feature.

For example, assume a tab is typed at the end of each prompt line:

```text
$ consul e
event  exec

$ consul r
reload  rtt

$ consul operator raft
list-peers   remove-peer
```

### Environment Variables <a id="environment-variables"></a>

In addition to CLI flags, Consul reads environment variables for behavior defaults. CLI flags always take precedence over environment variables, but it is often helpful to use environment variables to configure the Consul agent, particularly with configuration management and init systems.

These environment variables and their purpose are described below:

#### `CONSUL_HTTP_ADDR`

This is the HTTP API address to the _local_ Consul agent \(not the remote server\) specified as a URI with optional scheme:

```text
CONSUL_HTTP_ADDR=127.0.0.1:8500
```

or as a Unix socket path:

```text
CONSUL_HTTP_ADDR=unix://var/run/consul_http.sock
```

If the `https://` scheme is used, `CONSUL_HTTP_SSL` is implied to be true.

#### `CONSUL_HTTP_TOKEN` <a id="consul_http_token"></a>

This is the API access token required when access control lists \(ACLs\) are enabled, for example:

```text
CONSUL_HTTP_TOKEN=aba7cbe5-879b-999a-07cc-2efd9ac0ffe
```

#### `CONSUL_HTTP_AUTH` <a id="consul_http_auth"></a>

This specifies HTTP Basic access credentials as a username:password pair:

```text
CONSUL_HTTP_AUTH=operations:JPIMCmhDHzTukgO6
```

#### [»](https://www.consul.io/docs/commands/index.html#consul_http_ssl)`CONSUL_HTTP_SSL` <a id="consul_http_ssl"></a>

This is a boolean value \(default is false\) that enables the HTTPS URI scheme and SSL connections to the HTTP API:

```text
CONSUL_HTTP_SSL=true
```

#### `CONSUL_HTTP_SSL_VERIFY` <a id="consul_http_ssl_verify"></a>

This is a boolean value \(default true\) to specify SSL certificate verification; setting this value to `false` is not recommended for production use. Example for development purposes:

```text
CONSUL_HTTP_SSL_VERIFY=false
```

#### `CONSUL_CACERT` <a id="consul_cacert"></a>

Path to a CA file to use for TLS when communicating with Consul.

```text
CONSUL_CACERT=ca.crt
```

#### `CONSUL_CAPATH` <a id="consul_capath"></a>

Path to a directory of CA certificates to use for TLS when communicating with Consul.

```text
CONSUL_CAPATH=ca_certs/
```

#### `CONSUL_CLIENT_CERT` <a id="consul_client_cert"></a>

Path to a client cert file to use for TLS when `verify_incoming` is enabled.

```text
CONSUL_CLIENT_CERT=client.crt
```

#### `CONSUL_CLIENT_KEY` <a id="consul_client_key"></a>

Path to a client key file to use for TLS when `verify_incoming` is enabled.

```text
CONSUL_CLIENT_KEY=client.key
```

#### `CONSUL_TLS_SERVER_NAME` <a id="consul_tls_server_name"></a>

The server name to use as the SNI host when connecting via TLS.

```text
CONSUL_TLS_SERVER_NAME=consulserver.domain
```

#### `CONSUL_GRPC_ADDR` <a id="consul_grpc_addr"></a>

Like [`CONSUL_HTTP_ADDR`](https://www.consul.io/docs/commands/index.html#consul_http_addr) but configures the address the local agent is listening for gRPC requests. Currently gRPC is only used for integrating [Envoy proxy](https://www.consul.io/docs/connect/proxies/envoy.html) and must be [enabled explicitly](https://www.consul.io/docs/agent/options.html#grpc_port) in agent configuration.

```text
CONSUL_GRPC_ADDR=127.0.0.1:8502
```

or as a Unix socket path:

```text
CONSUL_GRPC_ADDR=unix://var/run/consul_grpc.sock
```

If the agent is [configured with TLS certificates](https://www.consul.io/docs/agent/encryption.html#rpc-encryption-with-tls), then the gRPC listener will require TLS and present the same certificate as the https listener. As with `CONSUL_HTTP_ADDR`, if TLS is enabled either the `https://` scheme should be used, or `CONSUL_HTTP_SSL` set.

