---
title: 应用层协议的优化
categories: SRE
date: 2020-08-29 13:10:28
tags: Nginx
---


## TLS/SSL 优化

### 握手性能优化

首先考虑 session cache，但是 session cache 只能在单台 Nginx 上使用。

```nginx
Syntax: ssl_session_cache off | none | [builtin[:size]] [shared:name:size];
Default: ssl_session_cache none; 
Context: http, server
```

<!-- more -->

- off：不使用 Session 缓存，且 Nginx 在协议中明确告诉客户端 Session 缓存不被使用
- none：不使用 Session 缓存，但是没有告诉客户端不被使用
- builtin：使用 OpenSSL 的 Session 缓存，由于在内存中使用，所以仅当统一客户端的两次连接都命中到同一个 worker 进程时，Session 缓存才会生效，比较少见
- shared:name:size：定义共享内存，为所有 worker 进程提供 Session 缓存服务，1MB 大于可用于 4000 个 Session。最常用

### TLS/SSL 中的会话票证 tickets

Nginx 将会话 Session 中的信息作为 tickets 加密发给客户端，当客户端下次发起 TLS 连接时，带上 tickets，由 Nginx 解密验证后复用会话 Session。

tickets 可以在集群中使用，因为只要集群的 Nginx 密钥相同即可。

会话票证虽然更易在 Nginx 集群中使用，但破坏了 TLS/SSL 的安全机制，有安全风险，必须频繁更换 tickets 密钥。

- 是否开启会话票证服务

```nginx
Syntax: ssl_session_tickets on | off;
Default: ssl_session_tickets on; 
Context: http, server
```

- 使用会话票证时加密 tickets 的密钥文件

```nginx
Syntax: ssl_session_ticket_key file;
Default: —
Context: http, server
```

## HTTP 长连接

- 减少握手次数
- 通过减少并发连接数减少了服务器资源的消耗
- 降低 TCP 拥塞控制的影响

```nginx
Syntax: keepalive_requests number;
Default: keepalive_requests 100; 
Context: http, server, location

Syntax: keepalive_requests number;
Default: keepalive_requests 100; 
Context: upstream
```

## gzip 压缩

- 功能：通过实时压缩 HTTP 包体，提升网络传输效率
- 模块：`ngx_http_gzip_module`，通过 `--without-http_gzip_module` 禁用模块

```nginx
Syntax: gzip on | off;
Default: gzip off; 
Context: http, server, location, if in location
```

### 压缩哪些请求的响应？

```nginx
Syntax: gzip_types mime-type ...;
Default: gzip_types text/html; 
Context: http, server, location

Syntax: gzip_min_length length;
Default: gzip_min_length 20; 
Context: http, server, location

Syntax: gzip_disable regex ...;
Default: —
Context: http, server, location

Syntax: gzip_http_version 1.0 | 1.1;
Default: gzip_http_version 1.1; 
Context: http, server, location
```

### 是否压缩上游的响应

```nginx
Syntax: gzip_proxied off | expired | no-cache | no-store | private | no_last_modified | no_etag | auth | any ...;
Default: gzip_proxied off; 
Context: http, server, location
```

- off：不压缩来自上游的响应
- expired：如果上游响应中含有 Expired 头部，且其值中的时间与系统时间比较后确定不会缓存，则压缩响应
- no-cache：如果上游响应中含有 Cache-Control 头部，且其值含有 no-cache 值，则压缩响应
- no-store：如果上游响应中含有 Cache-Control 头部，且其值含有 no-store值，则压缩响应
- private：如果上游响应中含有 Cache-Control 头部，且其值含有 private 值，则压缩响应
- no_last_modified：如果上游响应中没有 Last-Modified 头部，则压缩响应
- no_etag：如果上游响应中没有 Etag 头部，则压缩响应
- auth：如果客户端请求中含有 Authorization 头部，则压缩响应
- any：压缩所有来自上游的响应

### 其他压缩参数

还提供了一个变量，可以实时看到压缩率，例如输出到日志中。

```nginx
Syntax: gzip_comp_level level;
Default: gzip_comp_level 1; 
Context: http, server, location

Syntax: gzip_buffers number size;
Default: gzip_buffers 32 4k|16 8k; 
Context: http, server, location

Syntax: gzip_vary on | off;
Default: gzip_vary off; 
Context: http, server, location
```

## 升级更高的 HTTP/2 协议

向前兼容 HTTP/1.1 协议，传输效率可以大幅提升。

