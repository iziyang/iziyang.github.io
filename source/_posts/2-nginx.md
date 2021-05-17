---
title: Nginx 架构和基础原理
date: 2020-03-22 18:26:02
tags: Nginx
categories: SRE
---

# Nginx 的应用场景

Nginx 的应用场景主要有三个：

- 静态资源服务
- 反向代理服务
- API 服务

<!-- more -->

## 静态资源服务

Nginx 可以通过本地文件系统提供静态资源的服务，例如纯静态的 HTML 页面等。

## 反向代理服务

很多应用服务的运行效率是很低的，QPS，TPS，并发等都是受限的，所以需要把很多应用服务组成一个集群，向用户提供高可用性的服务，这个时候需要 Nginx 的反向代理功能，而应用服务的动态扩容需要负载均衡功能，另外一个，Nginx 层还需要做缓存。因此反向代理服务主要是三个功能：

- 反向代理
- 负载均衡
- 缓存

## API 服务

有时候应用服务本身有很多性能问题，但是数据库服务要比应用服务好的多，业务场景比较简单，并发性和 TPS 都要远高于应用服务，所以这个时候可以由 Nginx 直接去访问数据库或者 Redis，还可以利用 Nginx 的强大的并发性来实现应用防火墙的 API 服务。

# Nginx 架构基础

## Nginx 状态机

![](https://s3plus.sankuai.com/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200315065927)

Nginx 对外提供服务时，主要有三种流量会到达 Nginx：WEB、EMAIL、TCP 流量。这三种流量到达 Nginx 后，会分别由传输层状态机、应用层状态机、MAIL 状态机来处理。当内存不足以缓存所有的静态资源时，会退化成阻塞的磁盘调用，这个时候需要一个线程池来处理，对于每一个处理完的请求记录访问日志和错误日志，日志也是记录到磁盘中的。

## Nginx 的进程结构

![](https://s3plus.sankuai.com/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200315071337)

Nginx 有四种进程：

- Master 进程。Master 进程是父进程，其他进程都是子进程，Master 进程对 worker 进程进行管理
- worker 进程。worker 进程有多个，是负责处理具体的请求的。Nginx 为什么采用多进程而不是多线程的进程结构呢？是因为 Nginx 要保证高可用性，多线程之间会共享地址空间，当某一个第三方模块引发了一个段错误时，就会导致整个 Nginx 进程挂掉。而采用多进程模型不会出现这个问题
- cache manager 和 cache loader 进程。缓存除了要被多个 worker 进程使用，也要被 cache 进程使用，cache loader 做缓存的载入，cache manager 做缓存的管理，实际上每一个请求所使用的缓存还是由 worker 进程来进行的。这些进程间的通信都是通过共享内存来进行的

为什么 worker 进程需要很多个？

这是因为，Nginx 采用了事件驱动的模型后，它期望 worker 进程可以从头到尾占满一颗 CPU，这样可以更加高效的利用整颗 CPU，提高 CPU 的缓存命中率，另外还可以将 worker 进程与某一个 CPU 核绑定在一起。

## 使用信号管理 Nginx 的父子进程

在前面说到了 Nginx 的命令行，其实很多 Nginx 的信号都是通过向 master 进程发送信号来实现的。

### Master 进程

master 进程会监控 worker 进程，而监控是通过 Linux 规定的当子进程退出时需要向父进程发送 CHLD 信号实现的。这样可以当出现 bug 时，立刻拉起 worker。

master 进程可以接收以下信号：

- TERM, INT
- QUIT
- HUP
- USR1
- USR2
- WINCH

### Worker 进程

worker 进程可以接收以下信号：

- TERM, INT
- QUIT
- HUP
- USR1
- WINCH

### 命令行对应的信号

- reload：HUP
- reopen：USR1
- stop：TERM
- quit：QUIT

USR2 和 WINCH 没有对应的信号，只能通过 kill 发送。

stop 和 quit 的区别是，一个是立即退出，一个是优雅的停止。

### reload 重载配置文件的真相

1. 向 master 进程发送 HUP 信号
2. master 进程检查配置文件是否有语法问题
3. master 进程打开新的监听端口（如果配置了新的端口）
4. master 进程使用新的配置文件启动 worker 进程
5. master 进程向老的 worker 进程发送 QUIT 信号
6. 老 worker 进程关闭监听句柄，处理完当前连接后结束进程

![](https://s3plus.sankuai.com/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200315080228)

### 热部署的真相

在上一篇文章中，讲了热部署的流程，那么热部署具体的流程是怎么样的呢？

1. 将旧的 Nginx 文件替换成新的 Nginx 文件（注意备份）
2. 向 master 进程发送 USR2 信号
3. master 进程修改 pid 文件名，加后缀 .oldbin
4. master 进程用新 Nginx 文件启动新 master 进程
5. 向老的 master 进程发送 quit 信号，关闭老的 master
6. 回滚操作，向老的 master 进程发送 HUP 信号，向新的 master 发 QUIT

![](https://s3plus.sankuai.com/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200315080633)

### 优雅的关闭 worker 进程

1. 设置定时器 worker_shutdown_timeout
2. 关闭监听句柄
3. 关闭空闲连接
4. 再循环等待全部连接关闭
5. 退出进程

这里面，定时器的作用是，如果时间超时了，但是连接还没有处理完毕，就会强制退出进程。另外，Nginx 只能处理 HTTP 的优雅关闭，websocket 、TCP、UDP 的代理都做不到，worker 不解析数据。

以上这些内容，就是 Nginx 命令行和信号的完整过程。下一讲开始讲 HTTP 模块。