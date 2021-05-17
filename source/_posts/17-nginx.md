---
title: 七层反向代理对照
categories: SRE
date: 2020-08-29 12:47:49
tags: Nginx
---

之前详细讲了讲了 HTTP 的反向代理是如何实现的。这一篇文章来对比一下其他的应用层协议，看一下 Nginx 的指令。有这么四类协议，分别是 uwsgi、fastcgi、scgi。

<!-- more -->

## 七层反向代理对照

这个对照我们按照前面讲过的 HTTP 反向代理流程来做一一对比，实际上这些指令都是和 HTTP 反向代理的指令类似的，区别在于开头不同。有些指令没有是因为协议本身不支持。先来看构造请求内容。

### 构造请求内容

|  |uwsgi 反向代理|fastcgi 反向代理|scgi 反向代理|http 反向代理  |
| :-- | ------- | ------------- | -----------| -------------|
| 指定上游 | uwsgi_pass | fastcgi_pass | scgi_pass    | proxy_pass  |
| 是否传递请求头部 | uwsgi_pass_request_headers | fastcgi_pass_request_headers | scgi_pass_request_headers | proxy_pass_request_headers |
| 是否传递请求包体 | uwsgi_pass_request_body    | fastcgi_pass_request_body    | scgi_pass_request_body    | proxy_pass_request_body    |
|   指定请求方法名 |                            |                              |                           | proxy_method               |
|     指定请求协议 |                            |                              |                           | proxy_http_version         |
|   增、改请求头部 |                            |                              |                           | proxy_set_header           |
|     设置请求包体 |                            |                              |                           | proxy_set_body             |
| 是否缓存请求包体 | uwsgi_request_buffering    | fastcgi_request_buffering    | scgi_request_buffering    | proxy_request_buffering    |

这里面可以看到，指定请求方法名、指定请求协议、增改请求头部、设置请求包体这四类 uwsgi、fastcgi、scgi 是不需要这样的应用的，HTTP 则支持修改方法名、协议、头部和包体设置。

### 建立连接并发送请求

对于设置头部大小来说，uwsgi、fastcgi、scgi 没有此功能。

|                                | uwsgi 反向代理            | fastcgi 反向代理            | scgi 反向代理            | http 反向代理                  |
| :----------------------------- | ------------------------- | --------------------------- | ------------------------ | ------------------------------ |
| 连接上游超时时间               | uwsgi_connect_timeout     | fastcgi_connect_timeout     | scgi_connect_timeout     | proxy_connect_timeout          |
| 连接绑定地址                   | uwsgi_bind                | fastcgi_bind                | scgi_bind                | proxy_bind                     |
| 使用 TCP keepalive             | uwsgi_socket_keepalive    | fastcgi_socket_keepalive    | scgi_socket_keepalive    | proxy_socket_keepalive         |
| 忽略客户端关连接               | uwsgi_ignore_client_abort | fastcgi_ignore_client_abort | scgi_ignore_client_abort | proxy_ignore_client_abort      |
| 设置 HTTP 头部用到的哈希表大小 |                           |                             |                          | proxy_headers_hash_bucket_size |
| 设置 HTTP 头部哈希表最大值     |                           |                             |                          | proxy_headers_hash_max_size    |
| 发送请求超时时间               | uwsgi_send_timeout        | fastcgi_send_timeout        | scgi_send_timeout        | proxy_send_timeout             |

### 接收上游响应

再来看接收上游响应的部分，这一部分基本没有什么特别的。

|                    | uwsgi 反向代理             | fastcgi 反向代理             | scgi 反向代理             | http 反向代理              |
| :----------------- | -------------------------- | ---------------------------- | ------------------------- | -------------------------- |
| 是否缓存上游响应   | uwsgi_buffering            | fastcgi_buffering            | scgi_buffering            | proxy_buffering            |
| 存放上游响应的目录 | uwsgi_temp_path            | fastcgi_temp_path            | scgi_temp_path            | proxy_temp_path            |
| 写文件缓存大小     | uwsgi_temp_file_write_size | fastcgi_temp_file_write_size | scgi_temp_file_write_size | proxy_temp_file_write_size |
| 临时文件最大大小   | uwsgi_max_temp_file_size   | fastcgi_max_temp_file_size   | scgi_max_temp_file_size   | proxy_max_temp_file_size   |
| 接收响应头部缓存   | uwsgi_buffer_size          | fastcgi_buffer_size          | scgi_buffer_size          | proxy_buffer_size          |
| 接收响应包体缓存   | uwsgi_buffers              | fastcgi_buffers              | scgi_buffers              | proxy_buffers              |
| 缓存完成前转发包体 | uwsgi_busy_buffers_size    | fastcgi_busy_buffers_size    | scgi_busy_buffers_size    | proxy_busy_buffers_size    |
| 持久化包体文件     | uwsgi_store                | fastcgi_store                | scgi_store                | proxy_store                |
| 设定包体文件权限   | uwsgi_store_access         | fastcgi_store_access         | scgi_store_access         | proxy_store_access         |
| 读取响应超时时间   | uwsgi_read_timeout         | fastcgi_read_timeout         | scgi_read_timeout         | proxy_read_timeout         |
| 读取响应限速       | uwsgi_limit_rate           | fastcgi_limit_rate           | scgi_limit_rate           | proxy_limit_rate           |

### 转发响应

这一部分里面对于 Cookie 和修改 302 跳转来说，uwsgi、fastcgi、scgi 没有此功能。

|                              | uwsgi 反向代理              | fastcgi 反向代理              | scgi 反向代理              | http 反向代理               |
| :--------------------------- | --------------------------- | ----------------------------- | -------------------------- | --------------------------- |
| 减少发向客户端的响应头部     | uwsgi_hide_header           | fastcgi_hide_header           | scgi_hide_header           | proxy_hide_header           |
| 禁用响应头部功能             | uwsgi_ignore_headers        | fastcgi_ignore_headers        | scgi_ignore_headers        | proxy_ignore_headers        |
| 替换 Set-Cookie 头部中的域名 |                             |                               |                            | proxy_cookie_domain         |
| 替换 Set-Cookie 头部中的 URL |                             |                               |                            | proxy_cookie_path           |
| 修改重定向响应 Location 的值 |                             |                               |                            | proxy_redirect              |
| 传递头部到客户端             | uwsgi_pass_header           | fastcgi_pass_header           | scgi_pass_header           | proxy_pass_header           |
| 出错时更换上游               | uwsgi_next_upstream         | fastcgi_next_upstream         | scgi_next_upstream         | proxy_next_upstream         |
| 更换上游超时                 | uwsgi_next_upstream_timeout | fastcgi_next_upstream_timeout | scgi_next_upstream_timeout | proxy_next_upstream_timeout |
| 更换上游重试次数             | uwsgi_next_upstream_tries   | fastcgi_next_upstream_tries   | scgi_next_upstream_tries   | proxy_next_upstream_tries   |
| 拦截上游错误响应             | uwsgi_intercept_errors      | fastcgi_intercept_errors      | scgi_intercept_errors      | proxy_intercept_errors      |

### SSL

对于上游服务使用 SSL，fastcgi 和 scgi 是不支持的。

|                                       | uwsgi 反向代理        | fastcgi 反向代理 | scgi 反向代理 | http 反向代理         |
| :------------------------------------ | --------------------- | ---------------- | ------------- | --------------------- |
| 配置用于上游通讯的证书                | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 配置用于上游通讯的私钥                | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 指定安全套件                          | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 指定吊销证书链 CRL 文件验证上游的证书 | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 指定域名验证上游证书中的域名          | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 当私钥有密码时指定密码文件            | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 指定具体某个版本的协议                | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 传递 SNI 信息至上游                   | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 是否重用 SSL 连接                     | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 验证上游服务的证书                    | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 是否验证上游服务的证书                | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |
| 设置验证证书链的深度                  | uwsgi_ssl_certificate |                  |               | proxy_ssl_certificate |

### 缓存类指令

这里只有将 HEAD 方法转化为 GET 方法是不适用的。

|                                | uwsgi 反向代理                | fastcgi 反向代理                | scgi 反向代理                | http 反向代理                 |
| :----------------------------- | ----------------------------- | ------------------------------- | ---------------------------- | ----------------------------- |
| 指定共享内存                   | uwsgi_cache                   | fastcgi_cache                   | scgi_cache                   | proxy_cache                   |
| 缓存文件存放位置               | uwsgi_cache_path              | fastcgi_cache_path              | scgi_cache_path              | proxy_cache_path              |
| 指定哪些请求不使用缓存         | uwsgi_cache_bypass            | fastcgi_cache_bypass            | scgi_cache_bypass            | proxy_cache_bypass            |
| 开启子请求更新陈旧缓存         | uwsgi_cache_background_update | fastcgi_cache_background_update | scgi_cache_background_update | proxy_cache_background_update |
| 定义缓存关键字                 | uwsgi_cache_key               | fastcgi_cache_key               | scgi_cache_key               | proxy_cache_key               |
| 使用 range 协议的偏移          | uwsgi_cache_max_range_offset  | fastcgi_cache_max_range_offset  | scgi_cache_max_range_offset  | proxy_cache_max_range_offset  |
| 缓存哪些请求方法               | uwsgi_cache_methods           | fastcgi_cache_methods           | scgi_cache_methods           | proxy_cache_methods           |
| 多少请求后再缓存               | uwsgi_cache_min_uses          | fastcgi_cache_min_uses          | scgi_cache_min_uses          | proxy_cache_min_uses          |
| 缓存哪些响应及时长             | uwsgi_cache_valid             | fastcgi_cache_valid             | scgi_cache_valid             | proxy_cache_valid             |
| 强制使用 range 协议            | uwsgi_force_ranges            | fastcgi_force_ranges            | scgi_force_ranges            | proxy_force_ranges            |
| 有陈旧内容使用 304             | uwsgi_cache_revalidate        | fastcgi_cache_revalidate        | scgi_cache_revalidate        | proxy_cache_revalidate        |
| 返回陈旧的缓存内容             | uwsgi_cache_use_stale         | fastcgi_cache_use_stale         | scgi_cache_use_stale         | proxy_cache_use_stale         |
| 指定哪些响应不会写入缓存       | uwsgi_no_cache                | fastcgi_no_cache                | scgi_no_cache                | proxy_no_cache                |
| 将 HEAD 方法转换为 GET 方法    | uwsgi_                        | fastcgi_                        | scgi_                        | proxy_                        |
| 加锁减少回源请求               | uwsgi_cache_lock              | fastcgi_cache_lock              | scgi_cache_lock              | proxy_cache_lock              |
| 回源请求到达该超时时间后再放行 | uwsgi_cache_lock_age          | fastcgi_cache_lock_age          | scgi_cache_lock_age          | proxy_cache_lock_age          |
| 等待请求的最长等待时间         | uwsgi_cache_lock_timeout      | fastcgi_cache_lock_timeout      | scgi_cache_lock_timeout      | proxy_cache_lock_timeout      |

### 特殊的独有配置

这部分是这三个协议相对 HTTP 多出来的一些功能，这些功能也是很简单的。

|      | uwsgi 反向代理  | fastcgi 反向代理     | scgi 反向代理 | http 反向代理 |
| :--- | --------------- | -------------------- | ------------- | ------------- |
|      | uwsgi_modifier1 |                      |               |               |
|      | uwsgi_modifier2 |                      |               |               |
|      | uwsgi_param     | fastcgi_param        | uwsgi_param   |               |
|      |                 | fastcgi_index        |               |               |
|      |                 | fastcgi_catch_stderr |               |               |

## memcached 反向代理

- 功能：相对简单，只能转换 get 请求
  - 将 HTTP 请求转换为 memcached 协议中的 get 请求，转发请求至上游 memcached 服务
  - get 命令：get \<key>*\r\n
  - 控制命令：\<command name> \<key> \<flags> \<exptime> \<bytes> [noreply]\r\n
  - 通过设置 memcached_key 变量构造 key 键
- 模块：默认编译进 Nginx，ngx_http_memcached_module，通过 --without-http_memcached_module 禁用模块功能

### memcached 指令

memecached 是一个简化版的 HTTP 反向代理。

|                                                        | memcached 反向代理              | http 反向代理               |
| :----------------------------------------------------- | ------------------------------- | --------------------------- |
| 指定上游                                               | memcached_pass                  | proxy_pass                  |
| 连接绑定地址                                           | memcached_bind                  | proxy_bind                  |
| 接收响应头部缓存                                       | memcached_buffer_size           | proxy_buffer_size           |
| 连接上游超时时间                                       | memcached_connect_timeout       | proxy_connect_timeout       |
| 强制使用 range 请求                                    | memcached_force_ranges          | proxy_force_ranges          |
| 针对设置 key 时的 flag，对相应 flag 添加 gzip 响应头部 | memcached_gzip_flag             |                             |
| 出错时更换上游                                         | memcached_next_upstream         | proxy_next_upstream         |
| 更换上游超时                                           | memcached_next_upstream_timeout | proxy_next_upstream_timeout |
| 更换上游重试次数                                       | memcached_next_upstream_tries   | proxy_next_upstream_tries   |
| 读取响应超时时间                                       | memcached_read_timeout          | proxy_read_timeout          |
| 发送请求超时时间                                       | memcached_send_timeout          | proxy_send_timeout          |
| 使用 TCP keepalive                                     | memcached_socket_keepalive      | proxy_socket_keepalive      |

## WebSocket 反向代理

WebSocket 协议相对于 HTTP/1.1 协议而言，它可以从服务端向浏览器推送请求。另外，由于使用长连接、二进制的格式，因此 WebSocket 的通信效率比 HTTP/1.1 也要高一些。

- 功能：由 ngx_http_proxy_module 模块实现
- 配置：
  - proxy_http_version 1.1;
  - proxy_set_header Upgrade $http_upgrade;
  - proxy_set_header Connection "upgrade";

### 协议升级过程

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623064631)

- 客户端发送 HTTP 请求到服务器

在请求头里面需要标明：

```
Connection: keep-alive, Upgrade
Upgrade: websocket
```

- 服务器返回给客户端 101，表示支持 WebSocket

```
HTTP/1.1 101 Web Socket Protocol Handshake
```

### WebSocket 协议帧

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+
```

- FIN：共 1 位，标记消息是否是最后 1 帧，1 个消息由 1 个或多个数据帧构成，若消息由 1 帧构成，起始帧就是结束帧
- RSV1/RSV2/RSV3：各 1 位，预留位，用于自定义扩展。如果没有扩展，各位值为 0；如果定义了扩展，即为非 0 值；如果接收的帧中此处为 0，但是扩展中却没有该值的定义就关闭连接
- Payload len：7 位或 7+16 位或 7+64 位，如果值为：
  - 0-125：则 Payload Data 的长度即为该值
  - 126：那么接下来的 2 个字节才是 Payload Data 的长度（unsigned）
  - 127：那么接下来的 8 个字节才是 Payload Data 的长度（unsigned）
- opcode：4 位
  - 解释 Payload Data，如果接收到位置的 opcode，接收端必须关闭连接
  - 0x0 表示附加数据帧
  - 0x1 表示文本数据帧
  - 0x2 表示二进制数据帧
  - 0x3-7 暂时无定义，为以后的非控制帧保留
  - 0x8 表示连接关闭
  - 0x9 表示 ping，心跳请求
  - 0xA 表示 pong，心跳响应
  - 0xB-F 暂时无定义，为以后的控制帧保留
- MASK，共 1 位，掩码位，表示帧中的数据是否经过加密，客户端发出的数据帧需要经过掩码处理，这个值都是 1。如果值是 1，那么 Masking-key 域的数据就是掩码秘钥

### WebSocket 协议特点和扩展头部

- 数据分片有序。前一个 Frame 和后一个 Frame 之间存在依赖关系
- 不支持多路复用。由于有序，所以不支持多路复用
- 不支持压缩

#### 扩展头部

- Sec-WebSocket-Version：客户端发送，表示它想使用的 WebSocket 协议版本，如果服务器不支持发送的版本，就必须回应自己支持哪些版本
- Sec-WebSocket-Key：客户端发送，自动生成的一个键，作为一个对服务器的挑战，以验证服务器支持请求的协议版本
- Sec-WebSocket-Accept：服务器响应，包含 Sec-WebSocket-Accept 的签名值，证明它支持请求的协议版本
- Sec-WebSocket-Protocol：用于协商应用子协议，客户端发送支持的协议列表，服务器必须只回应一个协议名
- Sec-WebSocket-Extensions：用于协商本次连接要使用的 WebSocket 扩展，客户端发送支持的扩展，服务器通过返回相同的首部确认自己支持一个或多个扩展

## 实战

由于 WebSocket 的搭建比较麻烦，我们可以直接使用第三方搭建好的 WebSocket 服务来进行测试。