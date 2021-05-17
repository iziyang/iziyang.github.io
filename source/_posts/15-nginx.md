---
title: Nginx 的缓存
categories: SRE
date: 2020-08-29 12:47:01
tags: Nginx
---


## Nginx 缓存：定义存放缓存的载体

```nginx
Syntax: proxy_cache zone | off;
Default: proxy_cache off; 
Context: http, server, location

Syntax: proxy_cache_path path [levels=levels] [use_temp_path=on|off] 
        keys_zone=name:size [inactive=time] [max_size=size] [manager_files=number] 
        [manager_sleep=time] [manager_threshold=time] [loader_files=number] 
        [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] 
        [purger_sleep=time] [purger_threshold=time];
Default: —
Context: http
```

<!-- more -->

对于 Nginx 缓存来说，缓存的文件是在磁盘上的，而文件的元信息则是存放在内存中，所以需要先定义共享内存。

proxy_cache_path 指令需要定义缓存的路径以及共享内存中 key zone 的名字和大小，只能在 http 上下文中使用；而 proxy_cache 则可以使用 proxy_cache_path 中定义的 zone 名字。

下面介绍一下 proxy_cache_path 指令的含义。

## proxy_cache_path 指令

- path：定义缓存文件存放位置
- levels：定义缓存路径的目录层级，最多3 级，每层目录长度为 1 或者 2 字节
- use_temp_path
  - on：使用 proxy_temp_path 定义的临时目录
  - off 直接使用 path 路径存放临时文件
- keys_zone
  - name 是共享内存名字，由 proxy_cache 指令使用
  - size 是共享内存大小，1MB 大约可以存放 8000 个 key
- inactive：在 inactive 时间内没有被访问的缓存，会被淘汰掉，默认 10 分钟
- max_size：设置最大的缓存文件大小，超出后由 cache manager 进程按 LRU 链表淘汰
- manager_files：cache manager 进程在 1 次淘汰过程中，淘汰的最大文件数，默认 100
- manager_sleep：执行一次淘汰循环后 cache manager 进程的休眠时间，默认 200 毫秒
- manager_threshold：执行一次淘汰循环的最大耗时，默认 50 毫秒
- loader_files：cache loader 进程载入磁盘中缓存文件至共享内存，每批最多处理的文件数
- 默认 100
- loader_sleep：执行一次缓存文件至共享内存后，进程休眠的时间，载入默认 200 毫秒
- loader_threshold：每次载入缓存文件至共享内存的最大耗时，默认 50 毫秒

## 缓存的关键字

```nginx
Syntax: proxy_cache_key string;
Default: proxy_cache_key $scheme$proxy_host$request_uri; 
Context: http, server, location
```

## 缓存什么样的响应

这个指令非常重要，它用来表示我们缓存什么样的响应。

```nginx
Syntax: proxy_cache_valid [code ...] time;
Default: —
Context: http, server, location
```

- 对不同的响应码缓存不等的时长：例如：code 404 5m
- 只标识时间
  - 仅对以下响应码缓存
    - 200
    - 301
    - 302
- 通过响应头部控制缓存时长
  - X-Accel-Expires，单位秒
    - 为 0 时表示禁止 Nginx 缓存内容
    - 通过 @ 设置缓存到一天中的某一时刻
  - 响应头若含有 Set-Cookie 则不缓存
  - 响应头若含有 Vary 则不缓存

## 哪些内容不使用缓存？

- 参数为真时，响应不存入缓存。这里我们可以自己自定义一些变量或者使用其他的变量来设置是否缓存

```nginx
Syntax: proxy_no_cache string ...;
Default: —
Context: http, server, location
```

- 参数为真时，不使用缓存内容，当指定的 string 有值时，不使用缓存内容，直接发送到上游服务

```nginx
Syntax: proxy_cache_bypass string ...;
Default: —
Context: http, server, location
```

## 变更 HEAD 方法

这个指令默认会把 HEAD 方法转换为 GET 方法，如果设置为 off 则会继续使用 HEAD 方法。

```nginx
Syntax: proxy_cache_convert_head on | off;
Default: proxy_cache_convert_head on; 
Context: http, server, location
```

## upstream_cache_status 变量

- upstream_cache_status
  - MISS：未命中缓存
  - HIT：命中缓存
  - EXPIRED：缓存已经过期
  - STALE：命中了陈旧的缓存
  - UPDATING：内容陈旧，但正在更新
  - REVALIDATED：Nginx 验证了陈旧的内容依然有效
  - BYPASS：响应是从原始服务器获得的

### 实战

这里面配置文件与上一篇文章的配置文件相同。

```
➜  nginx curl cache.ziyang.com/ -I
HTTP/1.1 200 OK
Server: nginx/1.17.8
Date: Mon, 22 Jun 2020 14:45:49 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Mon, 25 May 2020 11:42:24 GMT
ETag: "5ecbaf20-264"
Expires: Mon, 22 Jun 2020 13:45:49 GMT
Cache-Control: no-cache
X-Cache-Status: MISS
Accept-Ranges: bytes
```

这里第一次访问可以看到 X-Cache-Status 是 MISS 的状态。再次访问：

```
➜  nginx curl cache.ziyang.com/ -I
HTTP/1.1 200 OK
Server: nginx/1.17.8
Date: Mon, 22 Jun 2020 14:46:41 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Mon, 25 May 2020 11:42:24 GMT
ETag: "5ecbaf20-264"
Expires: Mon, 22 Jun 2020 13:46:41 GMT
Cache-Control: no-cache
X-Cache-Status: HIT
Accept-Ranges: bytes
```

X-Cache-Status 已经是 HIT 状态了。

## 对客户端请求的缓存处理流程

### 缓存流程：发起请求部分



![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200622163329)

### 对哪个 method 方法使用缓存返回响应

```nginx
Syntax: proxy_cache_methods GET | HEAD | POST ...;
Default: proxy_cache_methods GET HEAD; 
Context: http, server, location
```

## 接收到上游响应后的缓存处理流程

### X-Accel-Expires 头部

```nginx
Syntax: X-Accel-Expires [offseconds]
Default: X-Accel-Expires off
```

从上游服务定义缓存多长时间。

- 0 表示不缓存当前响应
- @ 前缀表示缓存到当天的某个时间

### Vary 头部

- Vary: *：所有的请求都被视为唯一并且非缓存的，使用 Cache-Control: private 来实现则更适用，这样用于说明不存储该对象更加清晰，若没有通过 proxy_ignore_headers 设置忽略，则不缓存响应
- Vary: \<header-name>, \<header-name>, ...
  - 逗号分隔的一系列 HTTP 头部名称，用于确定缓存是否可用

### Set-Cookie 头部

```
Set-Cookie: <cookie-name>=<cookie-value> 
Set-Cookie: <cookie-name>=<cookie-value>; Expires=<date> 
Set-Cookie: <cookie-name>=<cookie-value>; Max-Age=<non-zero-digit> 
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain-value> 
Set-Cookie: <cookie-name>=<cookie-value>; Path=<path-value> 
Set-Cookie: <cookie-name>=<cookie-value>; Secure 
Set-Cookie: <cookie-name>=<cookie-value>; HttpOnly 
Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Strict 
Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Lax 
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain-value>; Secure; HttpOnly
```

若 Set-Cookie 头部没有被 proxy_ignore_headers 设置忽略，则不对响应进行缓存

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200622170403)

