---
title: 延迟关闭
categories: SRE
date: 2020-08-29 13:10:23
tags: Nginx
---


## lingering_close 延迟关闭的意义

当 Nginx 处理完成调用 close 关闭连接后，若接收缓冲区仍然收到客户端发来的内容，则服务器会向客户端发送 RST 包关闭连接，导致客户端由于收到 RST 而忽略了 response。

Nginx 发送 close 表示不希望再接收新的数据包了，而客户端如果仍然发送数据，就会直接回复 RST 包，客户端收到之后，就会认为是不正常的连接，从而忽略之前的响应内容。

<!-- more -->

### 配置指令

```nginx
Syntax: lingering_time time;
Default: lingering_time 30s; 
Context: http, server, location

Syntax: lingering_timeout time;
Default: lingering_timeout 5s; 
Context: http, server, location

Syntax: lingering_close off | on | always;
Default: lingering_close on; 
Context: http, server, location
```

- `lingering_timeout`：在 5s 内检测客户端是否仍然有请求内容到达，若超时后仍没有数据到达，就立刻关闭连接，如果有数据到达，那么就不会关闭连接，而是会等待到 `lingering_time` 后关闭
- `lingering_time`：整体的超时时间，达到这个时间后，不管有没有数据到达都会立刻关闭
- `lingering_close`
  - off：关闭功能
  - on：由 Nginx 判断，当用户请求未接收完（根据 chunk 或者 Content-Length 头部等）时启用功能，否则及时关闭连接
  - always：无条件启用功能

## 以 RST 代替正常的四次握手关闭连接

当其他读、写超时指令生效引发连接关闭时，通过发送 RST 立刻释放端口、内存等资源来关闭连接。这种情况很有可能是客户端异常了，所以需要立刻关闭连接。

```nginx
Syntax: reset_timedout_connection on | off;
Default: reset_timedout_connection off; 
Context: http, server, location
```

