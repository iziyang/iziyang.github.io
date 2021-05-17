---
title: 性能优化
categories: SRE
date: 2020-08-29 13:10:12
tags: Nginx
---


## 优化方法论

对 Nginx 的优化分为三个部分，首先优化 CPU，也就是进程调度模块，第二部分是网络层面的优化，第三部分是磁盘 IO 的性能优化。

<!-- more -->

- 从软件层面提升硬件使用效率
  - 增大 CPU 的利用率
  - 增大内存的利用率
  - 增大磁盘 IO 的利用率
  - 增大网络带宽的利用率
- 提升硬件规格
  - 网卡：万兆网卡
  - 磁盘：固态硬盘，关注 IOPS 和 BPS 指标
  - CPU：更快的主频，更多的核心，更大的缓存，更优的架构
  - 内存：更快的访问速度
- 超出硬件性能上限后使用 DNS 做集群的负载均衡

## 如何优化 CPU

这一部分看一下 Linux 进程间的调度是如何完成的。

### 如何增大 Nginx 使用 CPU 的有效时长？

想要增大 Nginx 使用 CPU 的有效时长，就需要保证 **Nginx 能够使用全部的 CPU 资源**，所以 worker 进程的数量应该大于或者等于 CPU 的核数，否则就会造成 CPU 不会总是调度 Nginx 进程。

- 能够使用全部 CPU 资源

  - master-worker 多进程架构
  - worker 进程数量应当大于等于 CPU 核数

**Nginx 进程间不做无用功浪费 CPU 资源**，不做无用功主要是指 worker 进程不应在繁忙时，主动让出 CPU，而这有两种情况：

- worker 进程间不应由于争抢造成资源耗散：worker 进程数量应当等于 CPU 核数
- worker 进程不应调用一些 API 导致主动让出 CPU：拒绝类似的第三方模块

同时还要保证 **Nginx 不被其他进程争抢资源**，这需要通过以下两点：

- 提升优先级占用 CPU 更长的时间
- 减少操作系统上耗资源的非 Nginx 进程

因此根据以上的方向可以进行以下配置：

- 设置 worker 进程的数量

```nginx
Syntax: worker_processes number | auto;
Default: worker_processes 1; 
Context: main
```

这里就引出了一个问题，那就是为什么一个 CPU 可以同时运行多个进程？

实际上在**宏观上是并行的，而在微观上是串行**的。

- 把进程的运行时间分为一段段的时间片
- OS 调度系统一次选择每个进程，最多执行时间片指定的时长

调度系统是根据进程的状态来选择进程的，如果进程处于运行态，那么就会放到排队器中进行排队，由 CPU 进行调度。

那么什么情况会导致进程主动让出 CPU 呢？

- 阻塞 API 引发的时间片内主动让出 CPU
  - 速度不一致引发的阻塞 API：例如硬件执行速度不一致，例如 CPU 和磁盘
  - 业务场景产生的阻塞 API：例如同步读网络报文

所以我们需要确保进程在**运行态**，进程有五种状态：

- R 运行：正在运行或在运行队列中等待
- S 中断：休眠中，受阻，在等待某个条件的行程或接受到信号
- D 不可中断：收到信号不唤醒和不可运行，进程必须等待直到有中断发生
- Z 僵死：进程已终止，但进程描述符存在，直到父进程调用 wait4() 系统调用后释放
- T 停止：进程收到 SIGSTOP、SIGSTP、SIGTIN、SIGTOU 信号后停止运行

### 减少进程间的切换

减少进程间的切换要保证下面几个部分： 

- Nginx worker 尽可能的处于 R 状态
  - R 状态的进程数量大于 CPU 核心时，负载急速升高
- 尽可能的减少进程间切换：进程间切换是指从一个进程切换到另一个进程或线程，前面说过，分为主动切换和被动切换，主动切换就是阻塞 API，被动切换则是时间片耗尽，这个时间片一般是小于 5 微秒
  - 减少主动切换：不调用阻塞 API
  - 减少被动切换：增大进程的优先级
- 绑定 CPU

还有一种方法是延迟处理新连接，因为新连接建立之后，还没有发送请求，这时候激活了 Nginx 也没有什么意义：

- 使用 TCP_DEFER_ACCEPT 延迟处理新连接

```nginx
Syntax: listen address[:port] [deferred];
Default: listen *:80 | *:8000; 
Context: server
```

### 如何查看上下文切换次数？

- vmstat
  - cs：表示上下文切换次数每秒钟切换次数
- dstat
  - csw：表示上下文切换次数
- pidstat -w -p
  - cswch/s：主动切换次数
  - nvcswch/s：被动切换次数

### 什么决定 CPU 时间片的大小

- Nice 静态优先级：-20~19，数字越小表示优先级越高
- priority 动态优先级：0~139

动态优先级是动态调整的，会根据是 CPU 消耗型进程还是 IO 消耗型进程来加上一定的幅度。在 2.6.23 版本之前，使用的是 O1 调度算法，幅度是加 -5，时间片在 5ms~800ms 之间，对于 Nginx worker 进程来说，我们是希望能够占据 800ms 的时间片的，对于 2.6.23 之后，是在静态优先级之上加 20。

所以 Nginx worker 进程的优先级一般会设置为最高 -20：

- 设置 worker 进程的静态优先级

```nginx
Syntax: worker_priority number;
Default: worker_priority 0; 
Context: main
```

## 多核间的负载均衡

这里记录一下，仅作参考。

### worker 进程间负载均衡

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626180944)

Linux 很早以前有一个 accept_mutex 指令，之前是默认打开的，现在是关闭的。打开主要是为了解决惊群问题，惊群问题是指我们可能有很多个 worker 进程，而当有请求过来的时候，只需要一个 worker 进程去处理，而不需要激活所有的 worker 进程，早期的 Linux 内核实现有问题，会导致所有的进程被激活，所以 accept_mutex 是在应用层加了一把锁，使得同一时间只有一个进程会被激活。

accept_mutex off 就是关闭了这种状态。

reuseport 是在内核层面提供了一个负载均衡策略，是在 Linux 3.0.90 之后引入的，这三种场景下的性能对比如上图所示。

从吞吐量来看 reuseport 有非常多的提升。 

### 多队列网卡对多核 CPU 的优化

- RSS：Receive Side Scaling，硬中断负载均衡，需要网卡支持，目前大部分网卡都支持这样的特性，之前硬中断可能在一颗 CPU 上执行，而现在可以发送到多颗 CPU 上在队列中同时执行
- RPS：Receive Packet Steering，软中断负载均衡
- RFS：Receive Flow Steering

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626210356)

这些特点并不是一定都要打开的，而是需要考虑具体情况，因为还是会消耗性能的。

### 提高 CPU 缓存命中率：worker_cpu_affinity

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626210508)

worker_cpu_affinity 是用来绑定 CPU 的。

```nginx
Syntax: worker_cpu_affinity cpumask ...;
        worker_cpu_affinity auto [cpumask];
Default: —
Context: main
```

CPU 是有缓存的，L1/L2/L3 三级缓存的大小是不同的，相应的延时也是不同的，如果经常在跨 CPU 进行访问的时候就相当于缓存就失效了。所以绑定 CPU 就可以提高缓存命中率。

这里涉及到一个 NUMA 架构下面来说一下。

### NUMA 架构

什么是 NUMA 架构呢？当 CPU 核心数越来越多，总线的数量却是一定的，这样当 CPU 访问的时候就会造成冲突。所以 NUMA 架构就是将总线分成两半，一半的核心数只能访问一半的内存，访问自己这一半的内存是很快的，而访问另外一半则很慢。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626211246)

它解决了冲突非常高的问题，但是带来了很多的问题。 例如当使用的内存超出了内存的一半的时候，就会导致访问 remote 的内存，这样也会导致命中率下降。 所以我们可以关闭访问远程，来解决这样一个问题。



