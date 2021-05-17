---
title: 对上游使用 SSL 连接
categories: SRE
date: 2020-08-29 12:45:58
tags: Nginx
---

我们大家都知道，在访问 Nginx 的时候，Nginx 可以提供证书供客户端验证，而实际上 Nginx 也可以验证客户端的证书，在连接上游服务器的时候，上游服务器也可以要求 Nginx 使用证书，也就是 Nginx 可以进行双向认证。

<!-- more -->

## 双向认证时的指令示例

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200622144902)

这张图有三个角色，客户端、反向代理 Nginx、上游服务 Nginx，反向代理和上游服务 Nginx 都可以分别指定自身使用的证书以及验证客户端的证书。

- 反向代理 Nginx

  - 指定自身使用的证书

    ```nginx
    proxy_ssl_certificate
    proxy_ssl_certificate_key
    ```

  - 验证服务器证书

    ```nginx
    proxy_ssl_verfiy
    proxy_ssl_trusted_certificate
    ```

- 上游服务 Nginx

  - 验证客户端证书

    ```nginx
    ssl_verfiy_client
    ssl_client_certificate
    ```

  - 指定自身使用的证书

    ```nginx
    ssl_certificate
    ssl_certificate_key
    ```

## 对下游使用证书

```nginx
Syntax: ssl_certificate file;
Default: —
Context: http, server

Syntax: ssl_certificate_key file;
Default: —
Context: http, server
```

注意这两个指令是不能放在 location 的，这个也是可以理解的，只能对 server 监听的端口使用证书。

## 验证下游证书

```nginx
Syntax: ssl_verify_client on | off | optional | optional_no_ca;
Default: ssl_verify_client off; 
Context: http, server

Syntax: ssl_client_certificate file;
Default: —
Context: http, server
```

## 对上游使用证书

```nginx
Syntax: proxy_ssl_certificate file;
Default: —
Context: http, server, location

Syntax: proxy_ssl_certificate_key file;
Default: —
Context: http, server, location
```

注意这里面出现了 location，因为只有在某一个 location 下连接上游服务的适合才需要去使用证书。

## 验证上游的证书

```nginx
Syntax: proxy_ssl_trusted_certificate file;
Default: —
Context: http, server, location

Syntax: proxy_ssl_verify on | off;
Default: proxy_ssl_verify off; 
Context: http, server, location
```

## SSL 模块提供的变量

### 安全套件

- ssl_cipher：本次通信选用的安全套件，例如 ECDHE-RSA-AES128-GCM-SHA256 

- ssl_ciphers：客户端支持的所有安全套件

- ssl_protocol：本次通信选用的 TLS 版本，例如 TLSv1.2

- ssl_curves：客户端支持的椭圆曲线，例如 secp384r1:secp521r1

### 证书

- ssl_client_raw_cert：原始客户端证书内容

- ssl_client_escaped_cert：返回客户端证书做 urlencode 编码后的内容

- ssl_client_cert：对客户端证书每一行内容前加 tab 制表符空白，增强可读性

- ssl_client_fingerprint：客户端证书的 SHA1 指纹

### 证书结构化信息

- ssl_server_name：通过 TLS 插件 SNI（Server Name Indication）获取到的服务域名
- ssl_client_i_dn：依据 RFC2253 获取到证书 issuer dn 信息，例如：CN=…,O=…,L=…,C=…
- ssl_client_i_dn_legacy：依据 8RFC2253 获取到证书 issuer dn 信息，例如：/C=…/L=…/O=…/CN=…
- ssl_client_s_dn：依据 RFC2253 获取到证书 subject dn 信息，例如：CN=…,OU=…,O=…,L=…,ST=…,C=…
- ssl_client_s_dn_legacy：同样获取 subject dn 信息，格式为：/C=…/ST=…/L=…/O=…/OU=…/CN=…

### 证书有效期

- ssl_client_v_end：返回客户端证书的过期时间，例如 Dec 1 11:56:11 2028 GMT
- ssl_client_v_remain：返回还有多少天客户端证书过期，例如针对上面的 ssl_client_v_end，其值为 3649
- ssl_client_v_start：客户端证书的颁发日期，例如 Dec 4 11:56:11 2018 GMT

### 连接有效性

- ssl_client_serial：返回连接上客户端证书的序列号，例如 8BE947674841BD44

- ssl_early_data：在 TLS1.3 协议中使用了 early data 且握手未完成返回 1，否则返回空字符串
- ssl_client_verify：如果验证失败为 FAILED，则这个值就是原因，如果没有验证证书则为“NONE”，验证成功则为 SUCCESS
- ssl_session_id：已经建立的 sessionid
- ssl_session_reused：如果 session 被复用（参考 session 缓存）则为 r，否则为 .

## 创建证书命令示例

- 创建根证书
  - 创建 CA 私钥：`openssl genrsa -out ca.key 2048`
  - 制作 CA 公钥：`openssl req -new -x509 -days 3650 -key ca.key -out ca.crt`
- 签发证书
  - 创建私钥：`openssl genrsa -out a.pem 1024`、`openssl rsa -in a.pem -out a.key`
  - 生成签发请求：`openssl req -new -key a.pem -out a.csr`
  - 使用 CA 证书进行签发：`openssl x509 -req -sha256 -in a.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out a.crt`
  - 验证签发证书是否正确：`openssl verify -CAfile ca.crt a.crt`