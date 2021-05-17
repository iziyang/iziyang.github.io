---
title: IO 调度算法
date: 2018-06-04 10:47:49
tags: Linux
categories: 编程
---

# IO 调度算法

## NOOP



NOOP 全称 No Operation，中文名称电梯式调度器，该算法实现了最简单的 FIFO 队列，所有 I/O 请求大致按照先来后到的顺序进行操作。之所以说”大致“，原因是 NOOP 在 FIFO 的基础上还做了相邻 IO 请求的合并，并不是完完全全按照先进先出的规则满足 IO 请求。它是基于先入先出（FIFO）队列概念的 Linux 内核里最简单的 I/O 调度器。此调度程序最适合于固态硬盘。

## Anticipatory

Anticipatory 的中文含义是"预料的，预想的"，顾名思义有个 I/O 发生的时候，如果又有进程请求 I/O 操作，则将产生一个默认的 6 毫秒猜测时间，猜测下一个进程请求 I/O 是要干什么的。这个 I/O 调度器对读操作优化服务时间，在提供一个 I/O 的时候进行短时间等待，使进程能够提交到另外的 I/O。Anticipatory 算法从 Linux 2.6.33 版本后被删除了，因为使用 CFQ 通过配置也能达到 Anticipatory 的效果。

## DeadLine

Deadline 翻译成中文是截止时间调度器，它避免有些请求太长时间不能被处理。另外可以区分对待读操作和写操作。DEADLINE 额外分别为读 I/O 和写 I/O 提供了 FIFO 队列。

## CFQ

CFQ 全称 Completely Fair Scheduler ，中文名称完全公平调度器，它是现在许多 Linux 发行版的默认调度器，CFQ 是内核默认选择的 I/O 调度器。该算法的特点是按照 IO 请求的地址进行排序，而不是按照先来后到的顺序来进行响应。CFQ 为每个进程/线程，单独创建一个队列来管理该进程所产生的请求，也就是说每个进程一个队列，各队列之间的调度使用时间片来调度，

# 相关命令

## 查看当前系统支持的I/O调度器

```shell
[sankuai@dx-cloud-climc01 ~]$ dmesg | grep -i scheduler
io scheduler noop registered
io scheduler anticipatory registered
io scheduler deadline registered
io scheduler cfq registered (default)
```

## 查看一个硬盘使用的I/O调度器

```Shell
[sankuai@gh-cloud-mss-store02 ~]$ cat /sys/block/sda/queue/scheduler
noop anticipatory deadline [cfq]
```

## 查看调度算法参数的含义

```
yum -y install kernel-doc
比如:/usr/share/doc/kernel-doc-2.6.32/Documentation/block/deadline-iosched.txt
```

## 永久修改系统 IO 调度器

### grubby 命令

```shell
$ grubby --grub --update-kernel=ALL --args="elevator=deadline
```

### 配置文件

```shell
使用vi编辑器修改grub配置文件
#vi cat /etc/default/grub
#修改第五行，在行尾添加#
elevator= cfq
 
然后保存文件，重新编译配置文件，
#grub2-mkconfig -o /boot/grub2/grub.cfg
```

# 算法调优

## CFQ 算法参数

```shell
[sankuai@gh-cloud-mss-store02 ~]$ ls -l /sys/block/sda/queue/iosched
total 0
-rw-r--r-- 1 root root 4096 Jun  4 11:53 back_seek_max
-rw-r--r-- 1 root root 4096 Jun  4 11:53 back_seek_penalty
-rw-r--r-- 1 root root 4096 Jun  4 11:53 fifo_expire_async
-rw-r--r-- 1 root root 4096 Jun  4 11:53 fifo_expire_sync
-rw-r--r-- 1 root root 4096 Jun  4 11:53 group_idle
-rw-r--r-- 1 root root 4096 Jun  4 11:53 group_isolation
-rw-r--r-- 1 root root 4096 Jun  4 11:53 low_latency
-rw-r--r-- 1 root root 4096 May 24  2017 quantum
-rw-r--r-- 1 root root 4096 Jun  4 11:53 slice_async
-rw-r--r-- 1 root root 4096 Jun  4 11:53 slice_async_rq
-rw-r--r-- 1 root root 4096 May 24  2017 slice_idle
-rw-r--r-- 1 root root 4096 Jun  4 11:53 slice_sync
```

- back_seek_max

  默认 16384。该参数规定了磁头向后寻址的最大范围,默认值是16M。对于请求所访问的扇区号在磁头后方的请求，cfq 会像向前寻址的请求一样调度他。

- back_seek_penalty

  默认值是 2。该参数用来计算向后寻址的代价。相对于前方查找，后方查找的距离为 1/2(1/back_seek_penalty)时，cfq 调度时就认为这两个请求寻址的代价是相同的。

- fifo_expire_async

  默认值是250ms。该参数用来控制异步请求的超时时间。如果队列被激活后，则优先检查是否有请求超时，如果有超时的请求，则派发。但是，在队列激活的期间内，只会派发 一个超时的请求，其余的请求按照请求的优先级，以及所访问的扇区号大小来派发。

- fifo_expire_sync

  默认值是125ms。功能类似于fifo_expire_async参数，该参数用于控制同步请求的超时时间，。

- group_idle

  默认值是 8ms，该参数为了提高吞吐量。

- group_isolation

  该参数用来标识应用程序所在的cgroup。

- low_latency

  该参数用来表示低延迟，默认值是1ms

- quantum

  该参数用于控制队列派发到设备驱动层所含有的请求数，默认值是8。不管是同步队列还是异步队列, 在时间片内，超过这个限制，则不再派发请求。对于异步队列 而言，请求数的派发个数还取决于参数slice_async_rq。

- slice_async

  默认值是 40ms。这个参数功能同slice_sync，但是用来计算异步队列的时间片。
  异步队列的时间片的计算公式是：time slice＝slice_async + (slice_async/5 * (4 - priority))；

- slice_async_rq

  默认值是 2。这个参数用来计算在时间片内异步请求被派发的最大数。同样，最大请求数也依赖于队列的优先级。计算公式是：最大请求数＝2 * slice_async_rq( 8 –priority );

- slice_idle

  默认值是 8ms，这个参数只控制同步队列的idle time。当同步队列当前没有请求派发时，并不切换到其他队列，而是等待 8ms，以便让应用程序产生更多的请求。直到同步队列的时间片用完。

- slice_sync

  默认值是 100ms。这个参数用来计算同步队列的时间片, 默认值是100ms。时间片还依赖于队列的优先级。同步队列的时间片的计算公式是：time slice＝slice_sync + (slice_sync/5 * (4 - priority))；

## Anticipatory 算法参数

```
The parameters are:
* read_expire
    Controls how long until a read request becomes "expired". It also controls the interval between which expired requests are served, so set to 50, a request might take anywhere < 100ms to be serviced _if_ it is the next on the expired list. Obviously request expiration strategies won't make the disk go faster. The result basically equates to the timeslice a single reader gets in the presence of other IO. 100*((seek time / read_expire) + 1) is very roughly the % streaming read efficiency your disk should get with multiple readers.

* read_batch_expire
    Controls how much time a batch of reads is given before pending writes are served. A higher value is more efficient. This might be set below read_expire if writes are to be given higher priority than reads, but reads are to be as efficient as possible when there are no writes. Generally though, it should be some multiple of read_expire.

* write_expire, and
* write_batch_expire are equivalent to the above, for writes.

* antic_expire
    Controls the maximum amount of time we can anticipate a good read (one with a short seek distance from the most recently completed request) before giving up. Many other factors may cause anticipation to be stopped early, or some processes will not be "anticipated" at all. Should be a bit higher for big seek time devices though not a linear correspondence - most processes have only a few ms thinktime.

In addition to the tunables above there is a read-only file named est_time
which, when read, will show:

    - The probability of a task exiting without a cooperating task
      submitting an anticipated IO.

    - The current mean think time.

    - The seek distance used to determine if an incoming IO is better.
```



## 测试-fio

可以使用专业的工具 fio，或者命令 dd。

### fio 格式

fio 可以采用命令行格式，也可以采用 jobfile 格式。

```shell
 fio [options] [jobfile] ...
   options
     --output=filename
```

### jobfile 参数

#### General

```shell
runtime=time 单位：seconds
```

#### I/O type

```shell
direct=bool # 是否使用缓存，true 表示不使用缓存
readwrite=str 
	read # 顺序读
	write # 顺序写
	randread # 随机读
	randwrite # 随即写
	rw # 顺序读写
	randrw # 随机读写
	
```

#### Block size

```Shell
blocksize/bs=int[,int][,int] # read, writes, trimes，默认是 4096
	bs=256k
blocksize_range/bsrange=irange[,irange][,irange]
	bsrange=1k-4k,2k-8k
bssplit=str
```

#### Buffers and memory

```shell
iomem=str, mem=str
	malloc
	shm
	shmhuge
	mmap
```

#### I/O size

```shell
size=int # 一个任务中每一个线程的总大小
io_size=int, io_limit=int
filesize=irange(int)
```

#### I/O engine

```shell
ioengine=str
	sync # 基本的读写操作
	psync # 基本的 pread 和 pwrite
	vsync # 基本的 readv 和 writev 将通过将相邻的 I/O 合并为一个提交来模拟排队。
	libaio # Linux 原生的异步 I/O
```

#### I/O depth

```shell
iodepth=int # 线程的数量
```

#### I/O rate

```shell
thinktime=time # I/O 发出后，在一段时间内停止，再发出下一个
rate=int[,int][,int] # 限制某一个任务的带宽
```

#### I/O latency

```shell
latency_target=time # 如果设置，fio将尝试查找给定工作负载运行时的最高性能点，同时保持低于此目标的延迟。
```

#### I/O replay

```shell
write_iolog=str
read_iolog=str
```

#### Threads, processes and job synchronization

```shell
thread # fio 默认是通过 fork 的方式创建任务，但是如果这个参数指定，就会采用 POSIX 的线程函数来创建线程
```

#### Verification

```shell
verify_only # 不指定工作负载，只验证数据是否符合之前的
```

#### Steady state

```shell
steadystate=str:float, ss=str:float # 定义评估稳态性能的标准和限制。
	iops # 收集 iops 数据 如：iops:2
```

#### Measurements and reporting

```shell
write_bw_log=str
write_lat_log=str
```

### 相关测试用例

#### 随机读

```shell
[Rand_Read_Testing]
direct=1
iodepth=128
rw=randread
ioengine=libaio
bs=4k
size=1G
numjobs=1
runtime=1000
group_reporting
filename=/s3plus/mount/002/randread.test
```

参考：http://devopslinux.com/2016/07/23/PTE-fio/

```shell
#顺序读
fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest

#顺序写
fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest

#随机读
fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest

#随机写
fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest

#混合随机读写
fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=100 -group_reporting -name=mytest -ioscheduler=noop
```

#### 混合随机读写

```shell
[randrw]
direct=1
iodepth=128
rw=randrw
ioengine=libaio
bs=4k
size=1G
numjobs=16
runtime=60
group_reporting
filename=/s3plus/mount/002/randrw.file
```







参考：

https://www.ibm.com/developerworks/cn/linux/l-lo-io-scheduler-optimize-performance/index.html

http://bbs.chinaunix.net/thread-1967733-1-1.html