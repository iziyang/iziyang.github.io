---
title: Linux 性能的优化
categories: SRE
date: 2020-08-29 13:10:36
tags: Nginx
---


## 磁盘 IO 的优化

**磁盘介质**

- 机械硬盘：一般适合大文件，例如日志
  - 价格低
  - 存储量大
  - BPS 较大，适用于顺序读写
  - IOPS 较小
  - 寿命长
- 固态硬盘：适合小文件的随机读写
  - 价格高
  - 存储量小
  - BPS 大
  - IOPS 大，适用于随机读写
  - 写寿命短

<!-- more -->

### 减少磁盘 IO

减少磁盘 IO 可以从以下几个方面来入手。

- 优化读取
  - sendfile 零拷贝
  - 内存盘、SSD 盘
- 减少写入
  - AIO
  - 增大 error_log 级别，减少写入的日志
  - 关闭 access_log：直接就不写日志
  - 压缩 access_log：减小写入日志的大小
  - 是否启用 proxy buffering
  - syslog 替代本地 IO
- 线程池 thread pool

![image-20200627213633025](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627213634)

### 直接 IO 绕开磁盘高速缓存

- 通常情况下，文件需要写入缓冲区然后再写到磁盘，读取的时候也是一样，先读到缓冲区，再读到用户存储层，相当于有两次拷贝
- 直接 IO 相当于绕开了缓冲区，直接从磁盘读写

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627213950)

### 直接 IO，适用于大文件

当磁盘上的文件大小超过 size 后，启用 directIO 功能，避免 Buffered IO 模式下磁盘页缓存中的拷贝消耗。

```nginx
Syntax: directio size | off;
Default: directio off; 
Context: http, server, location

Syntax: directio_alignment size;
Default: directio_alignment 512; 
Context: http, server, location
```

- directio：需要配置一个大小的指令，表示超过这个大小就会启用 directio。

### 异步 IO

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627214316)

在通常情况下，用户进程读取或者写入文件的时候会发生阻塞。而异步 IO 则是用户进程可以去处理其他任务，当写入或读取返回的时候再继续处理。

```nginx
Syntax: aio on | off | threads[=pool];
Default: aio off; 
Context: http, server, location

Syntax: aio_write on | off;
Default: aio_write off; 
Context: http, server, location
```

- aio：可以设置为 on，另外还可以设置线程池
- aio_write：通常情况下写入是写到磁盘高速缓存的，这个缓存是在内存中的，所以一般不需要开启 aio，而这个指令设置为 on 之后就会强制打开 aio，只有当接收上游服务的响应，同时开启了 proxy buffering，将响应写入到临时文件中的时候才有必要打开 aio

### 异步读 IO 线程池

- 编译时加 --with-threads

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627215005)

在静态资源服务场景下，就会有很多阻塞的场景，这时候需要线程池来处理这些请求。

```nginx
Syntax: thread_pool name threads=number [max_queue=number];
Default: thread_pool default threads=32 max_queue=65536; 
Context: main
```

- thread_pool：仅用于做静态资源服务的时候使用

### 异步 IO 中的缓存

将磁盘文件读入缓存中待处理，例如 gzip 模块会使用

```nginx
Syntax: output_buffers number size;
Default: output_buffers 2 32k; 
Context: http, server, location
```

## 减少磁盘读写次数

### empty_gif 模块

- 模块：`ngx_http_empty_gif_module` 模块，通过 `--without-http_empty_gif_module` 禁用模块
- 功能：从前端页面做用户行为分析时，由于跨域等要求，前端打点上报数据一般是 GET 请求，且考虑到浏览器解析 DOM 树的性能消耗，所以请求透明图片消耗最小，而 1*1 的 gif 图片体积最小（仅 43 字节），故通常请求 gif 图片，并在请求中把用户行为信息上报服务器
- Nginx 可以在 access 日志中获取到请求参数，进而统计用户行为，但若在磁盘中读取 1*1 的文件则有磁盘 IO 消耗，empty_gif 模块将图片放在内存中，加快了处理速度

```nginx
Syntax: empty_gif;
Default: —
Context: location
```

### access 日志的压缩

```nginx
Syntax: access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
Default: access_log logs/access.log combined; 
Context: http, server, location, if in location, limit_except
```

- buffer 默认 64 KB
- gzip 默认级别为 1
- 通过 zcat 解压查看

### error.log 日志输出内存

- 场景：在开发环境下定位问题时，若需要打开 debug 级别日志，但对 debug 级别大量日志引发的性能问题不能容忍，可以将日志输出到内存中
- 配置语法：`error_log memory:32m debug;`
- 查看内存中日志的方法：
  - gdb -p [worker 进程 id] -ex "source nginx.gdb" --batch
  - nginx.gdb 脚本内容

```nginx
set $log = ngx_cycle->log 
while $log->writer != ngx_log_memory_writer 
set $log = $log->next 
end 
set $buf = (ngx_log_memory_buf_t *) $log->wdata 
dump binary memory debug_log.txt $buf->start $buf->end
```

### syslog 协议

可以把针对磁盘 IO 的写操作通过网络写入到其他机器上面去。

网络协议：通过网络传输到 syslog 服务器

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627220326)

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627221225)

- server：定义 syslog 服务器地址

- facility
  - 参见RFC3164，取值：kern, user, mail, daemon, auth, intern, lpr, news,uucp, clock, authpriv, ftp, ntp, audit, alert, cron, local0..local7

- 默认是 locaI7

- severity：定义 access.log 日志的级别，默认 info 级别

- tag：定义日志的 tag，默认为 nginx

- nohostname：不向 syslog 中写入主机名 hostname

rsyslog 与 Nginx：使用 rsyslog 来搭建一个验证服务

## sendfile 零拷贝提升性能

- 减少进程间切换
- 减少内存拷贝次数

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627222418)

正常情况下，需要从内核缓冲区拷贝到应用程序缓冲区，而 sendfile 零拷贝技术就是不需要从内核拷贝到应用程序缓冲区而是直接从页缓存发送到 socket 缓冲区。

但是有一个问题，那就是如果开启了 sendfile，然后又需要做 gzip 压缩的时候，是必须读取到磁盘上的，这时候就产生了一个 gzip_static 模块。

## gzip_static 模块

这个模块会把静态文件在原有目录下做一下压缩，文件必须是以 .gz 结尾的，这样在 root 或者 alias 指令后面配置，会先检测是否有 gzip 文件，如果有就直接返回给客户端。

- 模块：`ngx_http_gzip_static_module`，通过 `--with-http_gzip_static_module` 启用模块
- 功能：检测到通明 .gz 文件时，response 中以 gzip 相关 header 返回 .gz 文件的内容

```nginx
Syntax: gzip_static on | off | always;
Default: gzip_static off; 
Context: http, server, location
```

- on 表示检测客户端是否支持 gzip
- always 表示无论客户端是否支持都返回

## gunzip 模块

- 模块：`ngx_http_gunzip_module`，通过 `--with-http_gunzip_module` 启用模块
- 功能：当客户端不支持 gzip 时，且磁盘上仅有压缩文件，则实时解压缩并将其发送给客户端

```nginx
Syntax: gunzip on | off;
Default: gunzip off; 
Context: http, server, location

Syntax: gunzip_buffers number size;
Default: gunzip_buffers 32 4k|16 8k; 
Context: http, server, location
```

## tcmalloc

- 更快的内存分配器
  - 并发能力强于 glibc：并发线程数越多，性能越好
- 减少内存碎片
- 擅长管理小块内存

http://goog-perftools.sourceforge.net/doc/tcmalloc.html

### 使用方式

- 编译 google perf tools，得到 tcmalloc 库： 

  https://github.com/gperftools/gperftools/releases

- 编译 Nginx 时，加入编译选项

  - C 语言生成可执行文件的两个步骤
  - 通过 configure 设定编译的额外参数：`--with-cc-opt=OPTIONS set additional C compiler options`
  - 通过 configure 设定链接的额外参数：`--with-ld-opt=OPTIONS set additional linker options`
  - `--with-ld-opt=-ltcmalloc`

## 使用 gperftools 定位 Nginx 性能问题

![image-20200627230039846](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627230040)

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627230125)

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627230138)

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200627230154)