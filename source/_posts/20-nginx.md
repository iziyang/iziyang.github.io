---
title: HTTP2 协议
categories: SRE
date: 2020-08-29 13:09:52
tags: Nginx
---


## HTTP2 的主要特性

- 传输数据量的大幅减少
  - 以二进制方式传输
  - 标头压缩：头部的数据量非常大
- 多路复用及相关功能
  - 消息优先级：可以对同一个页面的请求设置优先级，例如 CSS 和 js 等文件优先级较高
- 服务器消息推送
  - 并行推送：在同一条 TCP 连接上可以并行的推送多条消息

<!-- more -->

## HTTP2 核心概念

- 连接 Connection：1 个 TCP 连接，包含一个或者多个 Stream
- 数据流 Stream：一个双向通讯数据流，包含多个 Message
- 消息 Message：对应 HTTP1 中的请求或者响应，包含一条或者多条 Frame
- 数据帧 Frame：最小单位，以二进制压缩格式存放 HTTP1 中的内容

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623074445)

## 协议分层

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623074638)

## 多路复用

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623074751)

多路复用可以保证这条 TCP 连接永远处于最大带宽。因为 TCP 具有拥塞控制，而滑动窗口在刚开始的时候是很小的，所以频繁的创建连接会很消耗性能，而多路复用没有这个问题，工作效率最高，同时也减少了三次握手开关连接的消耗

## 传输中无序，接收时组装

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623075152)

在传输的时候是无序的，可以先传 stream 1，然后中间加入 stream 3，这些都没有问题，接收的时候重新组装到一起即可。

## 数据流优先级

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623075353)

- 数据流优先级可以用来保证数据流之间的依赖关系
- 每个数据流都有优先级（1-256）

## 标头压缩

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623075715)

当传输两个 message 的时候，只会传输不同的 message 头部，而不会把所有头部都传输一遍，大大提升了性能。

## 帧格式

```
 +-----------------------------------------------+
 |                 Length (24)                   |
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31)                      |
 +=+=============================================================+
 |                   Frame Payload (0...)                      ...
 +---------------------------------------------------------------+
```

Type 类型：

- HEADERS：帧仅包含 HTTP 标头信息
- DATA：帧包含消息的所有或部分有效负载
- PRIORITY：指定分配给流的优先级
- RST_STREAM：错误通知，一个推送承诺遭到拒绝。终止流
- SETTINGS：指定连接配置
- PUSH_PROMISE：通知一个将资源推送到客户端的意图
- PING：检测信号和往返时间
- GOAWAY：停止为当前连接生成流的停止通知
- WINDOW_UPDATE：用于管理流的流控制
- CONTINUATION：用于延续某一个标头碎片序列

## 服务器推送 PUSH

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623080934)

对于 js 和 css 文件可以直接推送给服务器。

下一节课来看一下 HTTP2 的这些优点是怎么用到的。

## 搭建 HTTP2 服务

- 模块：`ngx_http_v2_module`，通过 `--with-http_v2_module` 编译 Nginx 加入 HTTP2 协议的支持
- 功能：对客户端使用 HTTP2 协议提供基本功能
- 前提：开启 TLS/SSL 协议
- 使用方法：`listen 443 ssl http2;`

### Nginx 推送资源

```nginx
Syntax: http2_push_preload on | off;
Default: http2_push_preload off; 
Context: http, server, location

Syntax: http2_push uri | off;
Default: http2_push off; 
Context: http, server, location
```

添加头部：Link: resource_to_push

### 实战



- 测试 Nginx HTTP2 协议的客户端工具
  - 在 https://github.com/nghttp2/nghttp2/releases 下载安装

• centosyumyum install nghttp2

### 最大并行推送数

```nginx
Syntax: http2_max_concurrent_pushes number;
Default: http2_max_concurrent_pushes 10; 
Context: http, server
```

能够并发推送的资源个数，默认是 10。

### 超时控制

```nginx
Syntax: http2_recv_timeout time;
Default: http2_recv_timeout 30s; 
Context: http, server

Syntax: http2_idle_timeout time;
Default: http2_idle_timeout 3m; 
Context: http, server
```

多长时间没有收到请求就关闭连接

idle timeout 表示在指定时间之类没有请求也没有相应就关闭连接，默认是 3 分钟。

- 并发请求控制

```nginx
Syntax: http2_max_concurrent_pushes number;
Default: http2_max_concurrent_pushes 10; 
Context: http, server

Syntax: http2_max_concurrent_streams number;
Default: http2_max_concurrent_streams 128; 
Context: http, server

Syntax: http2_max_field_size size;
Default: http2_max_field_size 4k; 
Context: http, server
```

包括并发推送个数，以及在一条连接上并发的 stream 个数。

max_field_size 是做完压缩之后的 header 最大大小

- 连接最大请求处理数

```nginx
Syntax: http2_max_requests number;
Default: http2_max_requests 1000; 
Context: http, server

Syntax: http2_chunk_size size;
Default: http2_chunk_size 8k; 
Context: http, server, location
```

在一个 HTTP2 的连接最大处理的请求数量，默认是 1000

- 设置响应包体的分片大小

```nginx
Syntax: http2_chunk_size size;
Default: http2_chunk_size 8k; 
Context: http, server, location
```



- 缓冲区大小设置

```nginx
Syntax: http2_recv_buffer_size size;
Default: http2_recv_buffer_size 256k; 
Context: http

Syntax: http2_max_header_size size;
Default: http2_max_header_size 16k; 
Context: http, server

Syntax: http2_body_preread_size size;
Default: http2_body_preread_size 64k; 
Context: http, server
```

http2_recv_buffer_size 只针对每个 worker 进程，而不是每个 worker 连接

http2_max_header_size 对头部做完解压之后的大小

http2_body_preread_size 对每个请求而言预读的 body 的大小

## grpc 反向代理

- grpc 协议：https://grpc.io/
- 模块：`ngx_http_grpc_module`，通过 `--without-http_grpc_module` 禁用，依赖 `ngx_http_v2_module` 模块

### grpc 指令对照表

