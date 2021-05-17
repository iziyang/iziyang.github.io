---
title: 如何减轻缓存失效时上游服务的压力
categories: SRE
date: 2020-08-29 12:47:27
tags: Nginx
---


## 合并回源请求-减轻峰值流量下的压力

```nginx
Syntax: proxy_cache_lock on | off;
Default: proxy_cache_lock off; 
Context: http, server, location
```

<!-- more -->

同一时间，仅第一个请求发向上游，其他请求等待第 1 个响应返回或者超时后，使用缓存响应客户端。

```nginx
Syntax: proxy_cache_lock_timeout time;
Default: proxy_cache_lock_timeout 5s; 
Context: http, server, location
```

等待第 1 个请求返回响应的最大时间，到达后直接向上游发送请求，但不缓存响应。

```nginx
Syntax: proxy_cache_lock_age time;
Default: proxy_cache_lock_age 5s; 
Context: http, server, location
```

上一个请求返回响应的超时时间，到达后再放行一个请求发向上游。

## 减少回源请求——使用 stale 陈旧的缓存

虽然缓存已经过期，但是还是先使用旧缓存来返回给用户。

```nginx
Syntax: proxy_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | off ...;
Default: proxy_cache_use_stale off; 
Context: http, server, location
```

```nginx
Syntax: proxy_cache_background_update on | off;
Default: proxy_cache_background_update off; 
Context: http, server, location
```

直接启用 update，而不需要等待。

### proxy_cache_use_stale 指令：定义陈旧缓存的用法

- updating
  - 当缓存内容过期，有一个请求正在访问上游试图更新缓存时，其他请求直接使用过期内容返回客户端
  - stale-while-revalidate：缓存内容过期后，定义一段时间，在这段时间内 updating 设置有效，否则请求仍然访问上游服务，例如：Cache-Control: max-age=600, stale-while-revalidate=30
  - stale-if-error：缓存内容过期后，定义一段时间，在这段时间内上游服务出错后就继续使用缓存，否则请求仍然访问上游服务。stale-while-revalidate 包括 stale-if-error 场景
- error：当与上游建立连接、发送请求、读取响应头部等情况出错时，使用缓存
- timeout：当与上游建立连接、发送请求、读取响应头部等情况出现定时器超时，使用缓存
- http_(500|502|503|504|403|404|429)：缓存这些错误响应码的内容

### 缓存有问题的响应

```nginx
Syntax: proxy_cache_background_update on | off;
Default: proxy_cache_background_update off; 
Context: http, server, location
```

当使用 proxy_cache_use_stale 允许使用过期响应时，将同步生成一个子请求，通过访问上游服务更新过期的缓存。

```nginx
Syntax: proxy_cache_revalidate on | off;
Default: proxy_cache_revalidate off; 
Context: http, server, location
```

更新缓存时，使用 If-Modified-Since 和 If-None-Match 作为请求头部，预期内容未发生变更时通过 304 来减少传输的内容。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200622172502)

## 及时清除缓存

商业版中提供了这样的功能，免费版中需要使用一个第三方模块来做到。

- 模块：第三方模块 ngx_cache_purge：https://github.com/FRiCKLE/ngx_cache_purge，使用 --add-module= 指令添加模块到 Nginx 中
- 功能：接收到指定 HTTP 请求后立刻清除缓存

```nginx
syntax: proxy_cache_purge on|off|<method> [from all|<ip> [.. <ip>]] 
default: none 
context: http, server, location

syntax: proxy_cache_purge zone_name key 
default: none 
context: location
```

