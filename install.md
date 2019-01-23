# 安装指南

Installing Consul is simple. There are two approaches to installing Consul:

1. Using a [precompiled binary](https://www.consul.io/docs/install/index.html#precompiled-binaries)
2. Installing [from source](https://www.consul.io/docs/install/index.html#compiling-from-source)

Downloading a precompiled binary is easiest, and we provide downloads over TLS along with SHA256 sums to verify the binary. We also distribute a PGP signature with the SHA256 sums that can be verified.

### Precompiled Binaries <a id="precompiled-binaries"></a>

To install the precompiled binary, [download](https://www.consul.io/downloads.html) the appropriate package for your system. Consul is currently packaged as a zip file. We do not have any near term plans to provide system packages.

Once the zip is downloaded, unzip it into any directory. The `consul` binary inside is all that is necessary to run Consul \(or `consul.exe` for Windows\). Any additional files, if any, aren't required to run Consul.

Copy the binary to anywhere on your system. If you intend to access it from the command-line, make sure to place it somewhere on your `PATH`.

### Compiling from Source <a id="compiling-from-source"></a>

To compile from source, you will need [Go](https://golang.org/) installed and configured properly \(including a `GOPATH` environment variable set\), as well as a copy of [`git`](https://www.git-scm.com/) in your `PATH`.

1. Clone the Consul repository from GitHub into your `GOPATH`:

   ```text
   $ mkdir -p $GOPATH/src/github.com/hashicorp && cd $!
   $ git clone https://github.com/hashicorp/consul.git
   $ cd consul
   ```

2. Bootstrap the project. This will download and compile libraries and tools needed to compile Consul:

   ```text
   $ make tools
   ```

3. Build Consul for your current system and put the binary in `./bin/` \(relative to the git checkout\). The `make dev` target is just a shortcut that builds `consul` for only your local build environment \(no cross-compiled targets\).

   ```text
   $ make dev
   ```

### Verifying the Installation <a id="verifying-the-installation"></a>

To verify Consul is properly installed, run `consul -v` on your system. You should see help output. If you are executing it from the command line, make sure it is on your PATH or you may get an error about Consul not being found.

```text
$ consul -v
```

