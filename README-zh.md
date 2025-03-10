# rathole
![rathole-logo](./docs/img/rathole-logo.png)

[English](README.md) | [简体中文](README-zh.md)

安全、稳定、高性能的内网穿透工具，用 Rust 语言编写

rathole，类似于 [frp](https://github.com/fatedier/frp) 和 [ngrok](https://github.com/inconshreveable/ngrok)，可以让 NAT 后的设备上的服务通过具有公网 IP 的服务器暴露在公网上。

<!-- TOC -->

- [Features](#features)
- [Quickstart](#quickstart)
- [Configuration](#configuration)
  - [Logging](#logging)
- [Benchmark](#benchmark)
- [Development Status](#development-status)

<!-- /TOC -->

## Features

- **高性能** 具有更高的吞吐量，高并发下更稳定。见[Benchmark](#Benchmark)
- **低资源消耗** 内存占用远低于同类工具。见[Benchmark](#Benchmark)。[二进制文件最小](docs/build-guide.md)可以到 **~500KiB**，可以部署在嵌入式设备如路由器上。
- **安全性** 每个服务单独强制鉴权。Server 和 Client 负责各自的配置。使用 Noise Protocol 可以简单地配置传输加密，而不需要自签证书。同时也支持 TLS。
- **热重载** 支持配置文件热重载，动态修改端口转发服务。HTTP API 正在开发中。

## Quickstart

一个全功能的 `rathole` 可以从 [release](https://github.com/rapiz1/rathole/releases) 页面下载。或者 [从源码编译](docs/build-guide.md) 获取其他平台和裁剪的二进制文件。

`rathole` 的使用和 frp 非常类似，如果你有后者的使用经验，那配置对你来说非常简单，区别只是转发服务的配置分离到了服务端和客户端，并且必须要设置 token。

使用 rathole 需要一个有公网 IP 的服务器，和一个在 NAT 或防火墙后的设备，其中有些服务需要暴露在互联网上。

假设你在家里的 NAT 后面有一个 NAS，并且想把它的 ssh 服务暴露在公网上：

1. 在有一个公网 IP 的服务器上

创建 `server.toml`，内容如下，并根据你的需要调整。
```toml
# server.toml
[server]
bind_addr = "0.0.0.0:2333" # `2333` 配置了服务端监听客户端连接的端口

[server.services.my_nas_ssh]
token = "use_a_secret_that_only_you_know" # 用于验证的 token
bind_addr = "0.0.0.0:5202" # `5202` 配置了将 `my_nas_ssh` 暴露给互联网的端口
```

然后运行:
```bash
./rathole server.toml
```

2. 在 NAT 后面的主机（你的 NAS）上

创建 `client.toml`，内容如下，并根据你的需要进行调整。

```toml
# client.toml
[client]
remote_addr = "myserver.com:2333" # 服务器的地址。端口必须与 `server.bind_addr` 中的端口相同。
[client.services.my_nas_ssh]
token = "use_a_secret_that_only_you_know" # 必须与服务器相同以通过验证
local_addr = "127.0.0.1:22" # 需要被转发的服务的地址
```

然后运行：
```bash
./rathole client.toml
```

3. 现在 `rathole` 客户端会连接运行在 `myserver.com:2333`的 `rathole` 服务器，任何到 `myserver.com:5202` 的流量将被转发到客户端所在主机的 `22` 端口。

所以你可以 `ssh myserver.com:5202` 来 ssh 到你的NAS。

[Systemd examples](./examples/systemd) 中提供了一些让 `rathole` 在 Linux 上作为后台服务运行的配置示例。

## Configuration
如果只有一个 `[server]` 和 `[client]` 块存在的话，`rathole` 可以根据配置文件的内容自动决定在服务器模式或客户端模式下运行，就像 [Quickstart](#Quickstart) 中的例子。

但 `[client]` 和 `[server]` 块也可以放在一个文件中。然后在服务器端，运行 `rathole --server config.toml`。在客户端，运行 `rathole --client config.toml` 来明确告诉 `rathole` 运行模式。

**推荐首先查看 [examples](./examples) 中的配置示例来快速理解配置格式**，如果有不清楚的地方再查阅完整配置格式。

关于如何配置 Noise Protocol 和 TLS 来进行加密传输，参见 [Security](./docs/security.md)。

下面是完整的配置格式。
```toml
[client]
remote_addr = "example.com:2333" # Necessary. The address of the server
default_token = "default_token_if_not_specify" # Optional. The default token of services, if they don't define their own ones

[client.transport] # The whole block is optional. Specify which transport to use
type = "tcp" # Optional. Possible values: ["tcp", "tls", "noise"]. Default: "tcp"

[client.transport.tls] # Necessary if `type` is "tls"
trusted_root = "ca.pem" # Necessary. The certificate of CA that signed the server's certificate
hostname = "example.com" # Optional. The hostname that the client uses to validate the certificate. If not set, fallback to `client.remote_addr`

[client.transport.noise] # Noise protocol. See `docs/security.md` for further explanation
pattern = "Noise_NK_25519_ChaChaPoly_BLAKE2s" # Optional. Default value as shown
local_private_key = "key_encoded_in_base64" # Optional
remote_public_key = "key_encoded_in_base64" # Optional

[client.services.service1] # A service that needs forwarding. The name `service1` can change arbitrarily, as long as identical to the name in the server's configuration
type = "tcp" # Optional. The protocol that needs forwarding. Possible values: ["tcp", "udp"]. Default: "tcp"
token = "whatever" # Necessary if `client.default_token` not set
local_addr = "127.0.0.1:1081" # Necessary. The address of the service that needs to be forwarded

[client.services.service2] # Multiple services can be defined
local_addr = "127.0.0.1:1082"

[server]
bind_addr = "0.0.0.0:2333" # Necessary. The address that the server listens for clients. Generally only the port needs to be change. 
default_token = "default_token_if_not_specify" # Optional

[server.transport] # Same as `[client.transport]`
type = "tcp" 

[server.transport.tls] # Necessary if `type` is "tls"
pkcs12 = "identify.pfx" # Necessary. pkcs12 file of server's certificate and private key
pkcs12_password = "password" # Necessary. Password of the pkcs12 file

[server.transport.noise] # Same as `[client.transport.noise]`
pattern = "Noise_NK_25519_ChaChaPoly_BLAKE2s"
local_private_key = "key_encoded_in_base64" 
remote_public_key = "key_encoded_in_base64" 

[server.services.service1] # The service name must be identical to the client side
type = "tcp" # Optional. Same as the client `[client.services.X.type]
token = "whatever" # Necessary if `server.default_token` not set
bind_addr = "0.0.0.0:8081" # Necessary. The address of the service is exposed at. Generally only the port needs to be change. 

[server.services.service2] 
bind_addr = "0.0.0.1:8082"
```

### Logging
`rathole`，像许多其他 Rust 程序一样，使用环境变量来控制日志级别。

支持的 Logging Level 有 `info`, `warn`, `error`, `debug`, `trace`

比如将日志级别设置为 `error`:
```
RUST_LOG=error ./rathole config.toml
```

如果 `RUST_LOG` 不存在，默认的日志级别是 `info`。

## Benchmark

rathole 的延迟与 [frp](https://github.com/fatedier/frp) 相近，在高并发情况下表现更好，能提供更大的带宽，内存占用更少。

关于测试进行的更多细节，参见单独页面 [Benchmark](./docs/benchmark.md)。

![http_throughput](./docs/img/http_throughput.svg)
![tcp_bitrate](./docs/img/tcp_bitrate.svg)
![udp_bitrate](./docs/img/udp_bitrate.svg)
![mem](./docs/img/mem-graph.png)

## Development Status

`rathole` 正在积极开发中
- [x] 支持TLS
- [x] 支持UDP
- [x] 热重载
- [ ] 用于配置的HTTP APIs

[Out of Scope](./docs/out-of-scope.md) 列举了没有计划开发的特性并说明了原因。
