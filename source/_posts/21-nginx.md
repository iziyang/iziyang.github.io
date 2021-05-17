---
title: stream 四层反向代理的七个阶段和变量
categories: SRE
date: 2020-08-29 13:09:57
tags: Nginx
---


## stream 模块处理请求的 7 个阶段

| 阶段        | 模块                 |
| :---------- | -------------------- |
| POST_ACCEPT | realip               |
| PREACCESS   | limt_conn            |
| ACCESS      | access               |
| SSL         | ssl                  |
| PREREAD     | ssl_preread          |
| CONTENT     | return, stream_proxy |
| LOG         | access_log           |

<!-- more -->

## stream 中的 ssl

```nginx
Syntax: stream { ... }
Default: —
Context: main

Syntax: server { ... }
Default: —
Context: stream

Syntax: listen address:port [ssl] [udp] [proxy_protocol] [backlog=number] [rcvbuf=size] [sndbuf=size] [bind] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];
Default: —
Context: server
```

## 传输层相关的变量

- binary_remote_addr：客户端地址的整型格式，对于 IPv4 是 4 字节，对于 IPv6 是 16 字节
- connection：递增的连接序号
- remote_addr：客户端地址
- remote_port：客户端端口
- proxy_protocol_addr：若使用了 proxy_protocol 协议则返回协议中的地址，否则返回空
- proxy_protocol_port：若使用了 proxy_protocol 协议则返回协议中的端口，否则返回空
- protocol：传输层协议，值为 TCP 或者 UDP
- server_addr：服务器端地址
- server_port：服务器端端口
- bytes_received：从客户端接收到的字节数
- bytes_sent：已经发送到客户端的字节数
- status
  - 200：session 成功结束
  - 400：客户端数据无法解析，例如 proxy_protocol 协议的格式不正确
  - 403：访问权限不足被拒绝，例如 access 模块限制了客户端 IP 地址
  - 500：服务器内部代码错误
  - 502：无法找到或者连接上游服务
  - 503：上游服务不可用

## content 阶段：return 模块

```nginx
Syntax: return value;
Default: —
Context: server
```

### 实战



## proxy protocol 协议 与 realip 模块

v1 协议

- PROXY TCP4 202.112.144.236 10.210.12.10 5678 80\r\n 

- PROXY TCP6 2001:da8:205::100 2400:89c0:2110:1::21 6324 80\r\n 

- PROXY UKNOWN\r\n 

v2 协议

- 12 字节签名：\r\n\r\n\0\r\nQUIT\n
- 4 位协议版本号：2
- 4 位命令：0 表示 LOCAL，1 表示 PROXY，Nginx 仅支持 PROXY
- 4 位地址族：1 表示 IPv4，2 表示 IPv6
- 4 位传输层协议：1 表示 TCP，2 表示 UDP，Nginx 仅支持 TCP 协议
- 2 字节地址长度

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623160209)

### 读取 proxy_protocol 协议的超时控制

```nginx
Syntax: proxy_protocol_timeout timeout;
Default: proxy_protocol_timeout 30s; 
Context: stream, server
```

stream 处理 proxy_protocol 的流程：

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623160320)

## post_accept 阶段：realip 模块

- 功能：通过 proxy_protocol 协议取出客户端真实地址，并写入 remote_addr 及 remote_port 变量。同时使用 realip_remote_addr 和 realip_remote_port 保留 TCP 连接中获得的原始地址
- 模块：`ngx_stream_realip_module`，通过 `--with-stream_realip_module` 启用功能

```nginx
Syntax: set_real_ip_from address | CIDR | unix:;
Default: —
Context: stream, server
```

