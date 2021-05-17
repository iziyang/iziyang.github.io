---
title: Nginx 是如何处理 HTTP 头部的？
date: 2020-04-08 18:52:50
tags: Nginx
categories: SRE
---

# Nginx 处理 HTTP 头部的过程

Nginx 在处理 HTTP 请求之前，首先需要 Nginx 的框架先和客户端建立好连接，然后接收用户发来的 HTTP 的请求行，比如方法、URL 等，然后接收所有的 Header，根据这些 Header 信息，才能决定由哪些 HTTP 模块处理请求。下面这张图，解释了 Nginx 在处理 HTTP 请求之前，所经历的一系列流程，强烈建议收藏保存。下面针对每个部分单独讲解一下。

<!-- more -->

<img src="http://ww1.sinaimg.cn/large/c552abe7ly1gdpmcw4cx7j21uv3eeqee.jpg" alt="nginx 接收 HTTP 请求流程图.png"  />

# 接收请求事件模块

<img src="https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200412172435"  />

首先是三次握手，当客户端发来 ACK 之后，由操作系统内核回一个 SYN+ACK，紧接着客户端 ACK 之后，连接建立成功。同时可能有很多 worker 进程都在监听 80 或 443 端口，由操作系统的负载均衡算法，选取一个 worker 进程来处理，这个 worker 进程会通过 `epoll_wait` 方法，返回一个建立连接的句柄。拿到了监听的句柄之后，这实际上是一个读事件（因为是从操作系统中读取到了一个请求），调用 `accept` 方法，分配连接内存池。

> 内存池主要分为连接内存池和请求内存池。

连接内存池大小的配置是 `connection_pool_size`，到了这一步之后，Nginx 会为已经建立的连接分配一个 512 字节大小的连接内存池。分配完内存池，建立好连接之后，HTTP 模块会从事件模块手里接入请求处理的过程，HTTP 模块在启动时，会调用 `ngx_http_init_connection` 方法来设置回调方法，这个时候会把新建立连接的读事件通过 `epoll_ctl` 函数添加到 epoll 中，然后加一个超时定时器 `client_header_timeout: 60s`，这个定时器的作用是，如果超过 60s 还没有接收到客户端发来的请求，那么就会断开连接。这一部分走完之后，Nginx 的事件模块可能就会切换到其他的句柄去处理了。  

<img src="https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200412172508"  />

当用户真的把请求发来之后，操作系统会回复一个 ACK，同时事件模块的 `epoll_wait` 也拿到了这个请求，这个时候会调用设置的回调方法 `ngx_http_wait_request_handler`，将接收到的用户请求读到用户态中，而读取到用户态中需要操作系统分配内存，那么这段内存分配多大？从哪里分配呢？

这段内存是从连接内存池分配的，初始虽然分配了 512 字节，但是内存池可以扩展，由 `client_header_buffer_size: 1k` 分配 1k 内存，内存池并不是越大越好，因为用户即使发送了 1 个字节，也会分配出 1k 的内存出来。当 URL 超过 1k 后，应该怎么办呢？

# 接收请求 HTTP 模块

<img src="https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200412185833"  />

处理请求和处理连接是不一样的，处理请求只需要放到 Nginx 内存中就行了，但是处理请求还需要做大量的上下文分析，所以要分配一个请求内存池 `request_pool_size: 4k`。分配完以后，状态机开始解析请求行，如果这时候发现 URL 大于 4k，那么就会再分配一个大内存，也就是 `large_client_header_buffers: 4 8k`，这个配置的意思是说，最多分配 4 个 8k，它并不是一次性分配 32k，而是先分配 8k 然后再去解析请求行，如果依然大于 8k，那么就会再分配 8k 的内存。

Nginx 有很多变量，这些变量都是指针，其中可以用来标识 URI，标识完成之后，就开始处理 header。状态机解析 header 的时候，如果发现内存不够，也就是假如 URL 已经用掉了 `large_client_header_buffers: 4 8k` 中的 2 个 8k，这时候最多也只能分配 8k，请求行和 header 是公用 4 个 8k的。

分配完大内存之后，就开始标识 header，确定哪一个 server 块去处理请求，然后移除超时定时器，接下来，就开始核心的 11 个阶段 HTTP 请求处理请求。

这里需要注意以下几个地方：

- 连接内存池：初始大小 512 字节
  - `client_header_buffer_size: 1k` 从连接内存池中分配
  - `large_client_header_buffers: 4 8k` 也是从连接内存池中分配
- 请求内存池：`request_pool_size: 4k`

---

> 公众号「原少子杨」回复 Nginx 领取知识图谱

<img src="https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20191218223127" style="zoom:15%;" />