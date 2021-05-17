---
title: 想提升用户体验，那你知道怎么用好浏览器缓存吗？
categories: SRE
date: 2020-08-29 12:46:34
tags: Nginx
---

前面的文章详细描述了 Nginx 作为反向代理和负载均衡是如何处理请求的。在请求量小的情况下，我们不采取任何的优化措施问题似乎也不大，可是一旦请求量上来之后，优化性能就变得迫在眉睫了，而这里面非常重要的一个手段就是利用缓存。

<!-- more -->

## 浏览器缓存与 Nginx 缓存

**浏览器缓存**

- 优点
  - 使用有效缓存时，没有网络消耗，速度最快
  - 即使有网络消耗，但对失效缓存使用 304 响应做到网络流量消耗最小化
- 缺点
  - 仅提升一个用户的体验

**Nginx 缓存**

- 优点
  - 提升所有用户的体验
  - 相比浏览器缓存，有效降低上游服务的负载
  - 通过 304 响应减少 Nginx 与上游服务间的流量消耗
- 缺点
  - 用户仍然保持网络消耗

因此，通常我们需要同时使用浏览器与 Nginx 缓存来提升用户体验。

### 浏览器缓存

这是浏览器请求是否使用缓存的一个流程图，从这张图里面可以看出来有几个头部，分别是 Etag 和 Last-Modified、If-None-Match 和 If-Modified-Since。我先简单说一下这几个头部的意思，然后再来说一下流程。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200622153601)

### Etag 头部

Etag 响应头是资源的一个标识符，如果资源更改，那么一定要生成新的 Etag 值，所以 Etag 类似于指纹。Etag 还可以加上 'W/'，表示使用弱验证器，但不利于比较，强验证器是比较理想的选择，但很难有效的生成。对于相同资源的两个弱 Etag 值，可能语义相同，但不是每个字节都相同。

- etag 指令

```nginx
Syntax: etag on | off;
Default: etag on; 
Context: http, server, location
```

- 生成规则：这里其实就是一行代码，使用 - 分割前后两个部分，前半部分是使用的 last_modified 时间，后半部分则是包体长度

```
ngx_sprintf(etag->value.data, "\"%xT-%xO\"",
			r->headers_out.last_modified_time,
			r->headers_out.content_length_n)
```

### If-None-Match

If-None-Match 是一个条件请求头部，它是和 Etag 值进行比较的。

对于 GET 和 HEAD 请求方法来说，只有当验证失败，也就是 Etag 值匹配上了（注意是 None-Match，不匹配才是验证成功），必须返回 304 状态码。

当与 If-Modified-Since 同时存在的时候，If-None-Match 优先级更高。

### If-Modified-Since

If-Modified-Since 也是一个条件请求头部，只有当服务器请求的资源在给定的日期之后修改过，才会返回资源，状态码为 200，否则的话就返回 304。这个日期是根据 Last-Modified 头部来确定的。

还有一个头部叫 If-Unmodified-Since，和它不同的是 If-Modified-Since 只能用在 GET 或者 HEAD 请求中。

下面就来解释下上面的浏览器缓存的流程图。

1. 浏览器发起请求，如果有缓存的情况下判断缓存是否过期，缓存没有过期就直接从缓存读取，返回给用户，过期了需要请求服务器进行验证
2. 首先判断是否包含 Etag 头部，包含的话会向服务器发起携带 If-None-Match 头部的请求，然后根据文件内容是否更新返回 304 或者 200，如果不包含，那么接着进行判断
3. 继续判断是否包含 Last-Modified 头部，如果包含就携带 If-Modified-Since 发起请求，然后同样是判断文件是否更新返回 304 或者 200，否则就直接请求响应，然后进行缓存协商
4. 上面返回 304 的情况下，都会从 Nginx 缓存中读取内容

## Nginx 决策浏览器缓存是否过期

Nginx 是通过expires 指令来决策缓存是否过期的。下面来看一下。

### expires 指令

```nginx
Syntax: expires [modified] time;
        expires epoch | max | off;
Default: expires off; 
Context: http, server, location, if in location
```

- max：默认设置了一个非常大的数字

  - Expires: Thu, 31 Dec 2037 23:55:55 GMT
  - Cache-Control: max-age=315360000（10 年）

- off：不添加或者修改 Expires 和 Cache-Control 字段，默认也是 off。

- epoch：

  - Expires: Thu, 01 Jan 1970 00:00:01 GMT

  - Cache-Control：no-cache

- time：设定具体时间，可以携带单位

  - 一天内的具体时刻可以加 @，比如下午六点半：@18h30m
  - 设定好 Expires，自动计算 Cache-Control
  - 如果当前时间未超过当天的 time 时间，则 Expires 到当天 time，否则是第二天的 time 时刻
  - 正数：设定 Cache-Control 时间，计算出 Expires
  - 负数：Cache-Control: no-cache，计算出 Expires

### 实战

配置文件如下所示：

```nginx
proxy_cache_path /data/nginx/tmpcache levels=2:2 keys_zone=two:10m loader_threshold=300 
                     loader_files=200 max_size=200m inactive=1m;
server {
	server_name cache.ziyang.com;
	root html/;
	error_log logs/cacherr.log debug;
	location /{
		#expires @20h30m;
        expires 1h;
		#if_modified_since off;
		proxy_cache two;
		proxy_cache_valid 200 1m;
		add_header X-Cache-Status $upstream_cache_status;
		#proxy_cache_use_stale error timeout updating;
		#proxy_cache_key $scheme$uri;
		#proxy_cache_revalidate on;
		#proxy_cache_background_update on;
		#proxy_hide_header      Set-Cookie;
  		#proxy_ignore_headers   Set-Cookie;

		#proxy_force_ranges on;

		proxy_cache_key $scheme$uri;
		proxy_pass http://localhost:8012;
	}
    listen 80; # managed by Certbot
}
```

这里我先设置了 `expires 1h;` 看一下会返回什么：

```shell
➜  nginx curl cache.ziyang.com/testcache -I
HTTP/1.1 200 OK
Server: nginx/1.17.8
Date: Mon, 22 Jun 2020 14:16:50 GMT
Content-Type: text/plain
Content-Length: 136
Connection: keep-alive
Expires: Mon, 22 Jun 2020 15:16:50 GMT
Cache-Control: max-age=3600
X-Cache-Status: MISS
```

这里可以看到 Date 和 Expires 的时间恰好是差了 1 个小时，Cache-Control 的值是 3600。

如果设置成 `expires -1h;` 也可以再测试下：

```shell
➜  nginx curl cache.ziyang.com/testcache -I
HTTP/1.1 200 OK
Server: nginx/1.17.8
Date: Mon, 22 Jun 2020 14:18:56 GMT
Content-Type: text/plain
Content-Length: 136
Connection: keep-alive
Expires: Mon, 22 Jun 2020 13:18:56 GMT
Cache-Control: no-cache
X-Cache-Status: MISS
```

这里面 Expires 比 Date 要早一个小时，Cache-Control 为 no-cache。

## not_modified 过滤模块

- 功能：在客户端拥有缓存但是不确定缓存是否过期的情况下，会在请求中传入 If-Modified-Since 或者 If-None-Match 头部，该模块通过将其值与相应中的 Last-Modified 值相比较，决定是通过 200 返回全部内容，还是仅返回 304 Not Modified 头部
- 使用前提：原返回响应码为 200，也就是响应是存在的，而不是 403 或其他的一些异常状态码

这里面多了两个头部，一个是 If_Unmodified-Since，如果判断失败，也就是修改过，就会返回 412 错误，这一点是和 If-Modified-Since 不同的。

另外一个头部是 If_Match 头部，它同样是比较 Etag，与 If_None_Match 不同的是，使用强比较算法，只有在每一个比特都相同的情况下，才认为两个文件是相同的，在 Etag 前面添加 W/ 前缀表示可以使用相对宽松的算法。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200622155829)

下面来看一下 if_modified_since 指令的值。

```nginx
Syntax: if_modified_since off | exact | before;
Default: if_modified_since exact; 
Context: http, server, location
```

- off：忽略请求中的 if_modified_since 头部
- exact：精确匹配 if_modified_since 头部与 last_modified 的值
- before：若 if_modified_since 大于等于 last_modified 的值，则返回 304

直接演示一下：

```shell
curl -H 'If_Modified_Since: Sun, 11 Apr 2020 23:17:30 GMT' -H 'If_None_Match: "5e93a18a-264"' cache.ziyang.com/ -I
```

这条命令是返回 304 的。