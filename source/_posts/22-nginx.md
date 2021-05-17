---
title: 限并发连接、限 IP、记日志
categories: SRE
date: 2020-08-29 13:10:02
tags: Nginx
---


## PREACCESS 阶段的 limit_conn 模块

- 功能：限制客户端的并发连接数。使用变量自定义限制依据（例如根据客户端 IP 地址），基于共享内存所有 worker 进程同时生效
- 模块：默认编译进 `ngx_stream_limit_conn_module`，通过 `--without-stream_limit_conn_module` 禁用模块

<!-- more -->

### limit_conn 模块的指令

这个模块的指令与 HTTP 模块中的指令是非常相似的。

```nginx
Syntax: limit_conn_zone key zone=name:size;
Default: —
Context: stream

Syntax: limit_conn zone number;
Default: —
Context: stream, server

Syntax: limit_conn_log_level info | notice | warn | error;
Default: limit_conn_log_level error; 
Context: stream, server
```

- limit_conn_zone：与 HTTP 模块一样，首先要定义共享内存
- limit_conn：限制最大并发连接数
- limit_conn_log_level：当达到最大并发连接数后的记录日志级别，默认是 error

## ACCESS 阶段的 access 模块

- 功能：根据客户端地址（realip 模块可以修改地址）决定连接的访问权限
- 模块：`ngx_stream_access_module`，通过 `--without-stream_access_module` 禁用模块

### access 模块的指令

```nginx
Syntax: allow address | CIDR | unix: | all;
Default: —
Context: stream, server

Syntax: deny address | CIDR | unix: | all;
Default: —
Context: stream, server
```

与 HTTP 模块中一样，可以配置多条指令。

## log 阶段的 stream_log 模块

```nginx
Syntax: access_log path format [buffer=size] [gzip[=level]] [flush=time] [if=condition];
access_log off;
Default: access_log off; 
Context: stream, server

Syntax: log_format name [escape=default|json|none] string ...;
Default: —
Context: stream

Syntax: open_log_file_cache max=N [inactive=time] [min_uses=N] [valid=time];
open_log_file_cache off;
Default: open_log_file_cache off; 
Context: stream, server
```

首先需要定义日志的格式，`log_format` 指令，在具体的 stream 或 server 中可以通过 `access_log` 来定义模块的路径、buffer、gzip 等级，这些功能与 HTTP 模块中是完全一致的。

在 stream 中，是没有对文件的缓存的，所以这里只能用在日志上，对日志进行缓存，所以这里有一个专门的指令 `open_log_file_cache`。

## 实战



## stream 四层反向代理处理 SSL 下游流量

Nginx 作为四层反向代理也可以处理 SSL 协议的流量，可以分为三个场景：

- 客户端访问 Nginx 使用 TLS/SSL 协议，Nginx 转发到上游服务器同样是 TLS/SSL 协议
- 客户端访问 Nginx 使用 TLS/SSL 协议，Nginx 解析为 TCP 流量之后转发到上游
- 客户端访问 Nginx 使用 TCP 连接，Nginx 转发到上游通过 TLS/SSL 加密

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200625101559)

### stream 中的 ssl

- 功能：使 stream 反向代理对下游支持 TLS/SSL 协议
- 模块：`ngx_stream_ssl_module`，默认不编译进 Nginx，通过 `--with-stream_ssl_module` 加入

### stream ssl 指令对比 HTTP 模块

#### 配置基本参数

|                                                        | stream ssl                | http ssl                  |
| ------------------------------------------------------ | ------------------------- | ------------------------- |
| 配置服务器证书                                         | ssl_certificate           | ssl_certificate           |
| 配置服务器证书私钥                                     | ssl_certificate_key       | ssl_certificate_key       |
| 指定安全套件                                           | ssl_ciphers               | ssl_ciphers               |
| 指定吊销证书链 CRL                                     | ssl_crl                   | ssl_crl                   |
| 密码交换 DH 算法参数                                   | ssl_dhparam               | ssl_dhparam               |
| 指定密码交换时使用哪条椭圆曲线                         | ssl_ecdh_curve            | ssl_ecdh_curve            |
| TLS 握手的超时时间                                     | ssl_handshake_timeout     | ssl_handshake_timeout     |
| 指定私钥的密码文件                                     | ssl_password_file         | ssl_password_file         |
| 是否启用服务器偏好的安全套件                           | ssl_prefer_server_ciphers | ssl_prefer_server_ciphers |
| 指定支持的 TLS 协议                                    | ssl_protocols             | ssl_protocols             |
| 使用 session 缓存提升性能                              | ssl_session_cache         | ssl_session_cache         |
| 指定使用 ticket 票据时的加解密文件，默认使用随机字符串 | ssl_session_ticket_key    | ssl_session_ticket_key    |
| 是否启用 ticket 票据                                   | ssl_session_tickets       | ssl_session_tickets       |
| 指定 session 重用的超时时间                            | ssl_session_timeout       | ssl_session_timeout       |

#### 提升性能

|                                                        | stream ssl             | http ssl               |
| ------------------------------------------------------ | ---------------------- | ---------------------- |
| 使用 session 缓存提升性能                              | ssl_session_cache      | ssl_session_cache      |
| 指定使用 ticket 票据时的加解密文件，默认使用随机字符串 | ssl_session_ticket_key | ssl_session_ticket_key |
| 是否启用 ticket 票据                                   | ssl_session_tickets    | ssl_session_tickets    |
| 指定 session 重用的超时时间                            | ssl_session_timeout    | ssl_session_timeout    |

#### 验证客户端证书

|                                  | stream ssl              | http ssl                |
| -------------------------------- | ----------------------- | ----------------------- |
| 是否验证客户端的证书             | ssl_verify_client       | ssl_verify_client       |
| 可信 CA 证书，用于验证客户端证书 | ssl_client_certificate  | ssl_client_certificate  |
| 验证客户端证书是否可信           | ssl_trusted_certificate | ssl_trusted_certificate |
| 验证客户端证书链的深度           | ssl_verify_depth        | ssl_verify_depth        |

### stream ssl 模块提供的变量

与 HTTP 模块提供的变量也是完全一致的。

#### 安全套件

- ssl_cipher：本次通信选用的安全套件，例如 ECDHE-RSA-AES128-GCM-SHA256 

- ssl_ciphers：客户端支持的所有安全套件

- ssl_protocol：本次通信选用的 TLS 版本，例如 TLSv1.2

- ssl_curves：客户端支持的椭圆曲线，例如 secp384r1:secp521r1

#### 证书

- ssl_client_raw_cert：原始客户端证书内容

- ssl_client_escaped_cert：返回客户端证书做 urlencode 编码后的内容

- ssl_client_cert：对客户端证书每一行内容前加 tab 制表符空白，增强可读性

- ssl_client_fingerprint：客户端证书的 SHA1 指纹

#### 证书结构化信息

- ssl_server_name：通过 TLS 插件 SNI（Server Name Indication）获取到的服务域名
- ssl_client_i_dn：依据 RFC2253 获取到证书 issuer dn 信息，例如：CN=…,O=…,L=…,C=…
- ssl_client_s_dn：依据 RFC2253 获取到证书 subject dn 信息，例如：CN=…,OU=…,O=…,L=…,ST=…,C=…

#### 证书有效期

- ssl_client_v_end：返回客户端证书的过期时间，例如 Dec 1 11:56:11 2028 GMT
- ssl_client_v_remain：返回还有多少天客户端证书过期，例如针对上面的 ssl_client_v_end，其值为 3649
- ssl_client_v_start：客户端证书的颁发日期，例如 Dec 4 11:56:11 2018 GMT

#### 连接有效性

- ssl_client_serial：返回连接上客户端证书的序列号，例如 8BE947674841BD44
- ssl_client_verify：如果验证失败为 FAILED，则这个值就是原因，如果没有验证证书则为“NONE”，验证成功则为 SUCCESS
- ssl_session_id：已经建立的 sessionid
- ssl_session_reused：如果 session 被复用（参考 session 缓存）则为 r，否则为 .

### 实战

客户端这里通过裸的 TCP 协议向 Nginx 发送 HTTPS 请求，而 Nginx 向上游服务则是通过 HTTP 协议进行转发的。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626091449)

## SSL_PREREAD 模块

上面我们讲了当客户端发来 SSL 流到 Nginx，Nginx 需要剥离 SSL 加密信息并将 TCP 流转发给上游服务的一种情况。

还有另外一种情况是，当 Nginx 不需要剥离加密信息，只需要转发 SSL 数据流到上游服务器，Nginx 需要读取 SSL 握手信息时，来根据这些信息进一步选择上游服务器进行负载均衡。下面介绍一个处理这种场景的 SSL_PREREAD 模块。

- 模块：`stream_ssl_preread_module`，使用 `--with-stream_ssl_preread_module` 启用模块
- 功能：解析下游 TLS 证书中信息，以变量方式赋能其他模块。使用这个模块是不能使用 stream 模块的，也就是发向上游的一定还是 SSL 协议，但是可以把握手信息取出来

- 提供变量

  - $ssl_preread_protocol：客户端支持的 TLS 版本中最高的版本，例如 TLSv1.3
  - $ssl_preread_server_name：从 SNI 中获取到的服务器域名
  - $ssl_preread_alpn_protocols：通过 ALPN 中获取到的客户端建议使用的协议，例如 h2，http/1.1

### 指令

在看指令之前先来回顾一下 stream 模块处理请求的 7 个阶段：

| 阶段        | 模块                 |
| :---------- | -------------------- |
| POST_ACCEPT | realip               |
| PREACCESS   | limt_conn            |
| ACCESS      | access               |
| SSL         | ssl                  |
| PREREAD     | ssl_preread          |
| CONTENT     | return, stream_proxy |
| LOG         | access_log           |

PREREAD 阶段是在 SSL 阶段之后，如果开启了 SSL 阶段，那么当下游服务器经过 SSL 阶段处理成 TCP 流之后，在 PREREAD 阶段是取不到任何信息的。

```nginx
Syntax: preread_buffer_size size;
Default: preread_buffer_size 16k; 
Context: stream, server

Syntax: preread_timeout timeout;
Default: preread_timeout 30s; 
Context: stream, server

Syntax: ssl_preread on | off;
Default: ssl_preread off; 
Context: stream, server
```

- preread_buffer_size：我们需要解析 SSL 协议同时也需要原封不动的转发给上游，因此需要一段缓冲区
- preread_timeout：preread 也需要一个超时时间
- ssl_preread：是否打开功能，默认关闭

### 实战

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626113949)

## stream proxy 四层反向代理的用法

在 HTTP 模块中，客户端的真实 IP 地址可以通过 HTTP 头部传递过来，那么在四层反向代理中该怎么传呢？下面来看一下。

### stream_proxy 模块介绍

- 模块：`ngx_stream_proxy_module`，默认在 Nginx 中
- 功能：
  - 提供 TCP/UDP 协议的反向代理
  - 支持与上游的连接使用 TLS/SSL 协议
  - 支持与上游的连接使用 proxy protocol 协议

### proxy 模块对上下游的限速

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626115226)

对于图中的 4 条线来说，Nginx 只能对两条红色的线做限流，也就是对于接受数据流做限速，而不能对发送数据流做限速。这一点是因为，Nginx 作为四层反向代理内存缓存很小，不会像 HTTP 那样可以在磁盘缓存 body。所以如果限制住了客户端发来的上行流量也就限制了 Nginx 转发给上游服务的流量；限制了上游服务发来的下行流量也就限制了 Nginx 转发给客户端的流量。

- 限制读取上游服务数据的速度

```nginx
Syntax: proxy_download_rate rate;
Default: proxy_download_rate 0; 
Context: stream, server
```

- 限制读取客户端数据的速度

```nginx
Syntax: proxy_upload_rate rate;
Default: proxy_upload_rate 0; 
Context: stream, server
```

### 反向代理指令

|                                                        | stream 反向代理             | http 反向代理               |
| ------------------------------------------------------ | --------------------------- | --------------------------- |
| 指定上游                                               | proxy_pass                  | proxy_pass                  |
| 连接绑定地址                                           | proxy_bind                  | proxy_bind                  |
| 该值同时用于定义接收上游数据，接收下游数据的缓冲区大小 | proxy_buffer_size           | proxy_buffer_size           |
| 连接上游超时时间                                       | proxy_connect_timeout       | proxy_connect_timeout       |
| TCP 连接使用 proxy_protocol 协议                       | proxy_protocol              |                             |
| 出错时更换上游                                         | proxy_next_upstream         | proxy_next_upstream         |
| 更换上游超时                                           | proxy_next_upstream_timeout | proxy_next_upstream_timeout |
| 更换上游重试次数                                       | proxy_next_upstream_tries   | proxy_next_upstream_tries   |
| 读取响应超时时间                                       | proxy_timeout               | proxy_read_timeout          |
| 发送请求超时时间                                       | proxy_timeout               | proxy_send_timeout          |
| 使用 TCP keepalive                                     | proxy_socket_keepalive      | proxy_socket_keepalive      |

stream ssl 指令与 http proxy 模块对照表

|                            | stream 反向代理               | http 反向代理                 |
| -------------------------- | ----------------------------- | ----------------------------- |
| 是否对上游使用 ssl         | proxy_ssl                     |                               |
| 配置服务器证书             | proxy_ssl_certificate         | proxy_ssl_certificate         |
| 配置服务器证书私钥         | proxy_ssl_certificate_key     | proxy_ssl_certificate_key     |
| 指定安全套件               | proxy_ssl_ciphers             | proxy_ssl_ciphers             |
| 指定吊销证书链 CRL         | proxy_ssl_crl                 | proxy_ssl_crl                 |
| 指定域名验证上游证书中域名 | proxy_ssl_name                | proxy_ssl_name                |
| 当私钥有密码时指定密码文件 | proxy_ssl_password_file       | proxy_ssl_password_file       |
| 指定具体某个版本的协议     | proxy_ssl_protocols           | proxy_ssl_protocols           |
| 传递 SNI 信息至上游        | proxy_ssl_server_name         | proxy_ssl_server_name         |
| 是否重用 SSL 连接          | proxy_ssl_session_reuse       | proxy_ssl_session_reuse       |
| 验证上游服务的证书         | proxy_ssl_trusted_certificate | proxy_ssl_trusted_certificate |
| 是否验证上游服务的证书     | proxy_ssl_verify              | proxy_ssl_verify              |
| 设置验证证书链的深度       | proxy_ssl_verify_depth        | proxy_ssl_verify_depth        |

前面说过 proxy protocol 协议。这里做一个延时

### 实战

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200626154333)

