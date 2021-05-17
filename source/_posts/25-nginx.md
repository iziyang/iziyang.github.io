---
title: 网络层面优化
categories: SRE
date: 2020-08-29 13:10:18
tags: Nginx
---


## TCP 连接过程

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626212401)

<!-- more -->

实际任意一门编程语言都会有 connect、listen、write、close、read 等几个函数。另外使用 netstat 是可以看到这些状态的。

### SYN_SENT 状态

- `net.ipv4.tcp_syn_retries = 6`：主动建立连接时，发 SYN 的重试次数
- `net.ipv4.ip_local_port_range = 32768 60999`：建立连接时的本地端口可用范围

### 主动建立连接时应用层超时时间

```nginx
Syntax: proxy_connect_timeout time;
Default: proxy_connect_timeout 60s; 
Context: http, server, location

Syntax: proxy_connect_timeout time;
Default: proxy_connect_timeout 60s; 
Context: stream, server
```

### SYN_RCVD 状态

- `net.ipv4.tcp_max_syn_backlog`：SYN_RCVD 状态连接的最大个数，也就是接收 SYN 的数量是多少
- `net.ipv4.tcp_synack_retries`：被动建立连接时，发 SYN/ACK 的重试次数

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626213159)

这里面有两个队列：SYN 队列和 ACCEPT 队列，ACCEPT 队列中的连接都是已经成功建立的。也是有最大长度的，这个最大长度是 backlog。当 Nginx 被阻塞住的时候，队列也有可能被打满。

## 建立 TCP 连接优化

### 如何应对 SYN 攻击

攻击者短时间伪造不同 IP 地址的 SYN 报文，快速占满 backlog 队列，使得服务器不能正常为用户服务

- `net.core.netdev_max_backlog`：接收自网卡，但未被内核协议栈处理的报文队列长度
- `net.ipv4.tcp_max_syn_backlog`：SYN_RCVD 状态连接的最大个数
- `net.ipv4.tcp_abort_on_overflow`：超出处理能力时，对新来的 SYN 直接回包 RST，丢弃连接

第二种处理方案是采用 tcp_syncookies。

### tcp_syncookies

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626214351)

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626214424)

![image-20200626214508393](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626214508)

- `net.ipv4.tcp_syncookies = 1`：当 SYN 队列满后，新的 SYN 不进入队列，计算出 cookie 再以 SYN+ACK 中的序列号返回客户端，正常客户端发报文时，服务器根据报文中携带的 cookie 重新恢复连接
- 由于 cookie 占用序列号空间，导致此时所有的 TCP 可选功能失效，例如扩充窗口、时间戳等

### 一切皆文件：句柄数的上限

UNIX 操作系统中，一切皆文件，每一个文件都是一个文件句柄。

句柄数上线同样也限制了并发连接数。

- 操作系统全局：是对所有进程都生效的

  - fs.file-max：操作系统可使用的最大句柄数

  - 使用 fs.file-nr 可以查看当前已分配、正式用、上限：fs.file-nr = 21632 0 

    40000500

除了全局的还可以限制不同的登录用户：

- 限制用户
  - /etc/security/limits.conf
  - root soft nofile 65535：soft 是进程在运行中可以修改的连接数，soft 需要小于 hard
  - root hard nofile 65535：hard 是真实的连接数

最后一个是限制进程：

- 限制进程：限制 worker 进程可以达到的最大并发连接数

```nginx
Syntax: worker_rlimit_nofile number;
Default: —
Context: main
```

- 另外还有命令可以限制 worker 进程的最大连接数量、包括 Nginx 与上游下游间的连接

```nginx
Syntax: worker_connections number;
Default: worker_connections 512; 
Context: events
```

### 两个队列的长度

- SYN 队列未完成握手：`net.ipv4.tcp_max_syn_backlog = 262144`
- ACCEPT 队列已完成握手
  - `net.core.somaxconn`：系统级最大的 backlog 队列长度

同时 Nginx 还可以限制监听端口的最大 backlog 队列长度

```nginx
Syntax: listen address[:port] [backlog=number]
Default: listen *:80 | *:8000; 
Context: server
```

### TCP Fast Open

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626215918)

TCP Fast Open 是在第一次握手时服务端计算出一个 Cookie，然后发送给客户端。

第二次握手时直接携带 Cookie 和 HTTP 请求发送到服务端，服务端直接回 SYN+ACK 加 Data。实现这个功能需要客户端和服务端都支持。

- `net.ipv4.tcp_fastopen`：系统开启 TFO 功能
  - 0：关闭
  - 1：作为客户端时可以使用 TFO
  - 2：作为服务器可以使用 TFO
  - 3：无论作为客户端还是服务器，都可以使用 TFO

```nginx
Syntax: listen address[:port] [fastopen=number];
Default: listen *:80 | *:8000; 
Context: server
```

- fastopen=number：为防止带数据的 SYN 攻击，限制最大长度，指定 TFO 连接队列的最大长度

## 滑动窗口与缓冲区

前面介绍了 TCP 连接建立过程的指令以及优化方法。滑动窗口影响到了限速的指令以及 TCP 读写缓冲区的大小，通过设置缓冲区大小可以使网络更有效率。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626221417)

**功能**

- 用于限制连接的网速，解决报文乱序和可靠传输问题
- Nginx 中 limit_rate 等限速指令皆依赖它实现
- 由操作系统内核实现
- 连接两端各有发送窗口与接收窗口

### 发送 TCP 消息

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626221816)

### TCP 消息接收

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626221841)

### TCP 消息接收发生 CS

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626221912)

### TCP 消息接收时新报文到达

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626221944)

### Nginx 的超时指令与滑动窗口

- 两次读操作间的超时

```nginx
Syntax: client_body_timeout time;
Default: client_body_timeout 60s; 
Context: http, server, location
```

- 两次写操作间的超时

```nginx
Syntax: send_timeout time;
Default: send_timeout 60s; 
Context: http, server, location
```

- 读写超时

```nginx
Syntax: proxy_timeout timeout;
Default: proxy_timeout 10m; 
Context: stream, server
```

### 丢包重传

- 限制重传次数
  - `net.ipv4.tcp_retries1 = 3`：达到上限后，更新路由缓存
  - `net.ipv4.tcp_retries2 = 15`：达到上限后，关闭 TCP 连接
  - 仅作近似理解，实际以超时时间为准，可能少于重试次数就认为达到上限

## 如何优化 TCP 的缓冲区与传输效率

- `net.ipv4.tcp_rmem = 4096 87380 6291456`：读缓冲区最小值、默认值、最大值，单位字节，覆盖 `net.core.rmem_max`
- `net.ipv4.tcp_wmem = 4096 16384 4194304`：写缓冲区最小值、默认值、最大值，单位字节，覆盖 `net.core.wmem_max`
- `net.ipv4.tcp_mem = 1541646 2055528 3083292`：系统无内存压力、启动压力模式阈值、最大值，单位为页的数量
- `net.ipv4.tcp_moderate_rcvbuf = 1`：开启自动调整缓存模式

Nginx 也可以设置读写缓冲区，但是设置了之后就写死了，就不能使用 Linux 自动调整的这样的设置了。

```nginx
Syntax: listen address[:port] [rcvbuf=size] [sndbuf=size];
Default: listen *:80 | *:8000; 
Context: server
```

### 调整接收窗口与应用缓存

这个指令会把 TCP 缓冲区一分为二，一部分给应用缓存一部分给滑动窗口使用

- `net.ipv4.tcp_adv_win_scale = 1`：应用缓存=buffer / (2^tcp_adv_win_scale)

**BDP=带宽时延积**

滑动窗口大小=带宽*时延，也就是飞行中的报文大小

吞吐量=窗口/时延

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626223133)

### 禁用 Nagle 算法？

- Nagle 算法：避免一个连接上同时存在大量小报文
  - 最多只存在一个小报文
  - 合并多个小报文一起发送
- 提高带宽利用率

- 吞吐量优先：启用 Nagle 算法，`tcp_nodelay off`
- 低时延优先：禁用 Nagle 算法，`tcp_nodelay on`

```nginx
Syntax: tcp_nodelay on | off;
Default: tcp_nodelay on; 
Context: http, server, location
```

仅针对 HTTP KeepAlive 连接生效，默认禁用 Nagle 算法。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626223801)

Nginx 也可以避免发送小报文，只有当收集到了指定的字节数之后才会向客户端发送，如果报文已经是最后一个字节，那么就会立即发送出去。

```nginx
Syntax: postpone_output size;
Default: postpone_output 1460; 
Context: http, server, location
```

### 启用 CORK 算法？

仅针对 sendfile on 开启时有效，完全禁止小报文的发送，提升网络效率。

```nginx
Syntax: tcp_nopush on | off;
Default: tcp_nopush off; 
Context: http, server, location
```

## 慢启动与拥塞窗口

- 拥塞窗口：发送方主动限制流量，如果都按照接收方的最大接收窗口来的话，可能就会导致网络拥塞
- 通告窗口（对端接收窗口）：也就是滑动窗口，接收方限制流量，只考虑了点对点之间的网络问题
- 实际流量：拥塞窗口与通告窗口的最小值

### 拥塞处理

- 慢启动
  - 指数扩展拥塞窗口（cwnd 为拥塞窗口大小）
  - 每收到 1 个 ACK，cwnd=cwnd + 1
  - 每过一个 RTT，cwnd= cwnd*2
- 拥塞避免：窗口大于 threshold（阈值），开始采用线性扩展的方式
  - 线性扩展拥塞窗口
  - 每收到 1 个 ACK，cwnd=cwnd + 1/cwnd
  - 每过一个 RTT，窗口加 1
- 拥塞发生
  - 急速降低拥塞窗口，RENO 算法
  - RTO 超时，threshold=cwnd/2，cwnd=1
  - Fast Retransmit，收到 3 个 duplicate ACK，cwnd=cwnd/2，threshold=cwnd
- 快速恢复
  - 当 Fast Retransmit 出现时，cwnd 调整为 threshold + 3*MSS

### RTT 与 RTO

- RTT：Round Trip Time，表示链路上的总时延
  - 时刻变化
  - 组成
    - 物理链路传输时间
    - 末端处理时间
    - 路由器排队处理时间
  - 指导 RTO
- RTO：Retransmission TimeOut，正确的应用丢包，表示过多长时间需要重传

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626225614)

## TCP 的 Keep-Alive 功能

可以使服务器及时的剔除掉已经关闭的 TCP 连接，来释放资源，HTTP 中的 Keep-Alive 实际上是把短连接当做长连接来进行复用，TCP 的 Keep-Alive 则是为了及时的释放资源。

- 应用场景
  - 检测实际断掉的连接
  - 用于维持与客户端间的防火墙有活跃网络包

### Linux 的 TCP Keep-Alive 功能

- Linux 的 TCP Keep-Alive
  - 发送心跳周期：`net.ipv4.tcp_keepalive_time = 7200`，如果在 7200 秒之内没有收到任何的 TCP 报文，就会开启 Linux TCP 的检测功能，就是发送探测包，如果对端已经关闭了连接，那么操作系统就会发送一个 RESET 报文，如果网络原因没有收到回复，那么就会开始重试
  - 探测包发送间隔：`net.ipv4.tcp_keepalive_intvl = 75`
  - 探测包重试次数：`net.ipv4.tcp_keepalive_probes = 9`
- Nginx 的 TCP keepalive
  - so_keepalive=30m::10
  - keepidle, keepintvl, keepcnt，与上面的三个参数值一一对应

## 减少关闭连接时的 time_wait 端口数量

前面 TCP 连接的流程再来看一下断开连接的过程：

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626212401)

断开连接的时候，主动关闭的一方会进入 time_wait 状态。

### 被动关闭连接端的状态

- CLOSE_WAIT 状态：应用进程没有及时响应对端关闭连接
- LAST_ACK 状态：等待接收主动关闭端操作系统发来的针对 FIN 的 ACK 报文、

### 主动关闭连接端的状态

- fin_wait1 状态
  - `net.ipv4.tcp_orphan_retries = 0`：发送 FIN 报文的重试次数，0 相当于 8
- fin_wait2 状态
  - `net.ipv4.tcp_fin_timeout = 60`：保持在 FIN_WAIT_2 状态的时间
- time_wait 状态有什么用？

### TIME_WAIT 状态过短或者不存在会怎么样

- MSL（Maximum Segment Lifetime）：报文最大生存时间
- 维持 2MSL 时长的 TIME-WAIT 状态：保证至少一次报文的往返时间内端口是不可复用

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627180613)

过短的话，在下一次重建连接的时候如果又收到了 SEQ=3 的报文，就会发生混乱。

### TIME_WAIT 优化

- `net.ipv4.tcp_tw_reuse = 1`
  - 开启后，作为客户端时新连接可以使用仍然处于 TIME_WAIT 状态的端口
  - 由于 timestamp 的存在，操作系统可以拒绝迟到的报文
  - `net.ipv4.tcp_timestamps = 1`

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627181021)

这种场景下，假设客户端回复服务端的 ACK 丢失了，这时候 TIME_WAIT 比较短的话，由于可以重用处于 TIME_WAIT 状态的端口，这时候再发送 SYN 过去，而对端发送了 FIN,ACK，这时候就可以知道这是一个错误的报文，直接回复 RST 关闭连接，接下来就正常了。

- `net.ipv4.tcp_tw_recycle = 0`
  - 设置为 1 之后，表示同时作为客户端和服务器都可以使用 TIME_WAIT 状态的端口，这是不安全的，无法避免报文延迟、重复等给新连接造成混乱
- `net.ipv4.tcp_max_tw_buckets = 262144`
  - time_wait 状态连接的最大数量
  - 超出后直接关闭连接