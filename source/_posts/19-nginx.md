---
title: 使用 open file cache 提升系统性能
categories: SRE
date: 2020-08-29 13:09:46
tags: Nginx
---


## open_file_cache

```nginx
Syntax: open_file_cache off;
        open_file_cache max=N [inactive=time];
Default: open_file_cache off; 
Context: http, server, location
```

<!-- more -->

- 功能：默认是关闭的。当设置为 max 时，表示最多缓存多少个文件在内存中，而不是共享内存。因为只针对 worker 进程即可，如果超出了 max 设置的 N 的数量，就会使用 LRU 链表淘汰策略，inactive 表示过期时间，例如超出 30 秒没有访问就移出缓存

## 缓存信息

那么都会缓存哪些内容呢？主要有下面几类：

- 文件句柄：缓存了文件句柄就意味着不需要每次都重新 open 或者 close 文件
- 文件修改时间
- 文件大小
- 文件查询时的错误信息：例如 403 的错误信息
- 目录是否存在

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623071933)

## 其他 open_file_cache 指令

```nginx
Syntax: open_file_cache_errors on | off;
Default: open_file_cache_errors off; 
Context: http, server, location

Syntax: open_file_cache_min_uses number;
Default: open_file_cache_min_uses 1; 
Context: http, server, location

Syntax: open_file_cache_valid time;
Default: open_file_cache_valid 60s; 
Context: http, server, location
```

- open_file_cache_errors：表示对于文件错误的是否需要缓存，默认是 off
- open_file_cache_min_uses：至少要访问多少次，才会留在内存中
- open_file_cache_valid：表示过期时间，避免当文件修改了之后，Nginx 还是指向的旧的文件句柄

## 实战



