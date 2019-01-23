# bootstrap

Command: `consul acl bootstrap`

The `acl bootstrap` command will request Consul to generate a new token with unlimited privileges to use for management purposes and output its details. This can only be done once and afterwards bootstrapping will be disabled. If all tokens are lost and you need to bootstrap again you can follow the bootstrap reset procedure.

The ACL system can also be bootstrapped via the [HTTP API](https://www.consul.io/api/acl/acl.html#bootstrap-acls).

### Usage <a id="usage"></a>

Usage: `consul acl bootstrap [options]`

**API Options**

* * [`-ca-file=<value>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#ca-file-lt-value-gt-) - Path to a CA file to use for TLS when communicating with Consul. This can also be specified via the `CONSUL_CACERT` environment variable.
* [`-ca-path=<value>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#ca-path-lt-value-gt-) - Path to a directory of CA certificates to use for TLS when communicating with Consul. This can also be specified via the `CONSUL_CAPATH` environment variable.
* [`-client-cert=<value>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#client-cert-lt-value-gt-) - Path to a client cert file to use for TLS when `verify_incoming` is enabled. This can also be specified via the `CONSUL_CLIENT_CERT` environment variable.
* [`-client-key=<value>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#client-key-lt-value-gt-) - Path to a client key file to use for TLS when `verify_incoming` is enabled. This can also be specified via the `CONSUL_CLIENT_KEY` environment variable.
* [`-http-addr=<addr>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#http-addr-lt-addr-gt-) - Address of the Consul agent with the port. This can be an IP address or DNS address, but it must include the port. This can also be specified via the `CONSUL_HTTP_ADDR` environment variable. In Consul 0.8 and later, the default value is [http://127.0.0.1:8500](http://127.0.0.1:8500/), and https can optionally be used instead. The scheme can also be set to HTTPS by setting the environment variable `CONSUL_HTTP_SSL=true`. This may be a unix domain socket using`unix:///path/to/socket` if the [agent is configured to listen](https://www.consul.io/docs/agent/options.html#addresses) that way.
* [`-tls-server-name=<value>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#tls-server-name-lt-value-gt-) - The server name to use as the SNI host when connecting via TLS. This can also be specified via the `CONSUL_TLS_SERVER_NAME` environment variable.
* [`-token=<value>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#token-lt-value-gt-) - ACL token to use in the request. This can also be specified via the `CONSUL_HTTP_TOKEN`environment variable. If unspecified, the query will default to the token of the Consul agent at the HTTP address.
* * [`-datacenter=<name>`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#datacenter-lt-name-gt-) - Name of the datacenter to query. If unspecified, the query will default to the datacenter of the Consul agent at the HTTP address.
* [`-stale`](https://www.consul.io/docs/commands/acl/acl-bootstrap.html#stale) - Permit any Consul server \(non-leader\) to respond to this request. This allows for lower latency and higher throughput, but can result in stale data. This option has no effect on non-read operations. The default value is false.

The output looks like this:

```text
AccessorID:   4d123dff-f460-73c3-02c4-8dd64d136e01
SecretID:     86cddfb9-2760-d947-358d-a2811156bf31
Description:  Bootstrap Token (Global Management)
Local:        false
Create Time:  2018-10-22 11:27:04.479026 -0400 EDT
Policies:
   00000000-0000-0000-0000-000000000001 - global-management
```

