---
title: UDP 反向代理
categories: SRE
date: 2020-08-29 13:10:07
tags: Nginx
---


## UDP 反向代理

在实时音视频这样的场景中，UDP 协议更加适合，但是在这种实时的场景下，不会再考虑过期的视频帧或音频帧，UDP 少了很多的协议头部，性能也会更高一些。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626155248)

<!-- more -->

UDP 有一个 session 的概念，当从客户端发送消息到 Nginx 再到上游服务时，这中间的一系列端口就建立了一个 session。

UDP 反向代理中有下面两个指令。

```nginx
Syntax: proxy_requests number;
Default: proxy_requests 0; 
Context: stream, server
```

- 指定一次会话 session 中最多从客户端接收到多少报文就结束 session（1.15.7 非稳定版本）
  - 仅会话结束时才会记录 access 日志
  - 同一个会话中，Nginx 使用同一端口连接上游服务
  - 设置为 0 表示不限制，每次请求都会记录 access 日志

```nginx
Syntax: proxy_responses number;
Default: —
Context: stream, server
```

- 指定对应一个请求报文，上游应返回多少个响应报文
  - 与 proxy_timeout 结合使用，控制上游服务是否不可用

## 透传 IP 地址

经常我们需要透传客户端的 IP 地址来进行处理，透传 IP 地址有多种方案，下面分别介绍一下。

- proxy_protocol 协议

- 修改 IP 报文

第一种就是前面讲过的利用 proxy_protocol 协议；第二种是修改 IP 报文，下面主要说一下修改 IP 报文的方案。

**步骤**

  - 修改 IP 报文中的源地址
  - 修改路由规则

修改 IP 报文中的源地址后，还需要修改上游服务的路由规则，因为原本的 IP 地址是 Nginx 的，修改了源地址之后就不会再发往 Nginx 了，因此需要修改路由规则，基于此就产生了两种方案：

- IP 地址透传：经由 Nginx 转发上游返回的报文给客户端（TCP/UDP）
- DSR：上游直接发送报文给客户端（仅 UDP）

第一种方案就是上面说的修改路由规则，仍然通过 Nginx 转发报文；第二种方案则是如果公网是通的，那么可以直接发送报文给客户端，这个仅针对 UDP。

下面分别说一下这两种方案。

### IP 地址透传

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626163321)

- proxy_bind $remote_addr transparent;

- 调节上游服务所在主机上的网关为 Nginx 所在主机

  ```shell
  # route del default gw 10.0.2.2（原网关 IP）
  # route add default gw 172.16.0.1（Nginx 所在主机的 IP）
  ```

- 调节 Nginx 所在主机上的路由规则，使它把接收自上游的、目标 IP 是客户端 IP 的报文转发给 Nginx

  ```shell
  ip rule add fwmark 1 lookup 100
  ip route add local 0.0.0.0/0 dev lo table 100
  iptables -t mangle -A PREROUTING -p tcp -s 172.16.0.0/28 --sport 80 -j MARK --set-xmark 0x1/0xffffffff
  ```

### DSR 上游服务直接向客户端回包

由上游服务直接将报文发送给客户端。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626164131)

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626164144)

- 加 `proxy_responses 0`

- `proxy_bind $remote_addr:$remote_port transparent;`

- 若上游服务在内网无公网 IP，则可由 Nginx 所在主机转发

  - 在上游服务所在主机添加路由

    ```nginx
     route add default gw nginx-ip-address
    ```

- 允许操作系统转发 IP 报文

  ```
   sysctl -w net.ipv4.ip_forward=1
  ```

- 转发时修改源地址为 Nginx 所在主机的地址

  ![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626164716)

直接向客户端回包会出现两个问题：

- Nginx 检测不到上游服务是否回包
- 负载均衡策略受限

Nginx 的负载均衡策略是需要通过检测上游服务是否回包来决定的，如果上游服务不回包了，自然也就检测不到了。