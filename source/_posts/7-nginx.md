---
title: Nginx 的变量究竟是怎么一回事？
tags: Nginx
categories: SRE
date: 2020-06-09 06:11:12
---

之前说了很多关于 Nginx 模块的内容，还有一部分非常重要的内容，那就是 Nginx 的变量。变量在 Nginx 中可以说无处不在，认识了解这些变量的作用和原理同样是必要的，下面几乎囊括了关于 Nginx 的所有变量，单独看起来可能比较枯燥，放心，后面依然有实战内容。

<!-- more -->

## Nginx 变量的运行原理

围绕 Nginx 中的变量模块可以分为两类，一类是提供变量的模块，另外一类是使用变量的模块。

- 提供变量的模块
  - 在 Preconfiguration 源代码中定义变量名以及可以解析出变量的方法
- 使用变量的模块
  - 解析 nginx.conf 时定义变量的使用方式

也就是在 Nginx 启动时，已经定义了变量，而只有当真正处理请求的时候，才会根据 nginx.conf 解析出来的变量使用方式调用 Preconfiguration 中定义的方法来实际获取值。

这也是变量的两个特性：

- 惰性求值：只有使用的时候才会去调方法解析
- 变量值可以时刻变化，其值为使用的那一时刻的值。例如发送响应包体字节数，实际在发送的过程中是一直在变化的。

除了 Nginx 的模块之外，Nginx 框架也包含许多的变量，这些变量不需要通过编译模块来引入，而且，Nginx 框架所提供的变量往往反映了处理请求的细节，因此，了解 Nginx 框架所提供的变量是十分有必要的。

## HTTP 请求相关的变量

先来看一下关于 HTTP 请求的相关变量。

- arg_参数名：URL 中某个具体参数的值

- query_string：与 args 变量完全相同

- args：全部 URL 参数

- is_args：如果请求 URL 中有参数则返回 ?，否则返回空

- content_length：HTTP 请求中标识包体长度的 Content-Length 头部的值。如果请求中没有携带这个参数，那么就取不到对应的值。

- content_type：标识请求包体类型的 Content-Type 头部的值。同样需要用户请求中携带对应的参数。

- uri：请求的 URI（不同于 URL，不包括 ? 后的参数）

- document_uri：与 uri 完全相同。由于历史原因而存在的。

- request_uri：请求的 URL（包括 URI 以及完整的参数）

- scheme：协议名，例如 HTTP 或者 HTTPS

- request_method：请求方法，例如 GET 或者 POST

- request_length：所有请求内容的大小，包括请求行、头部、包体等

- remote_user：由 HTTP Basic Authentication 协议传入的用户名

- request_body_file：很多时候会将用户请求的包体存放到文件中，这个变量就是临时存放请求包体的文件

  - 如果包体非常小则不会存文件
  - client_body_in_file_only 指令强制所有包体存入文件，且可决定是否删除

- request_body：请求中的包体，这个变量当且仅当使用反向代理，且设定用内存暂存包体时才有效

- request：原始的 URL 请求，含有方法与协议版本，例如 GET /?a=1&b=22 HTTP/1.1

- host

  - 先从请求行中获取
  - 如果含有 Host 头部，则用其值替换掉请求行中的主机名
  - 如果前两者都取不到，则使用匹配上的 server_name

- http_头部名字：返回一个具体请求头部的值

  特殊变量，这些变量会做一些处理。

  - http_host
  - http_user_agent
  - http_referer
  - http_via
  - http_x_forwarded_for
  - http_cookie

  通用变量，除了以上的变量，都可以取到对应的值。

## TCP 连接相关的变量

下面是关于 TCP 连接的变量。

- binary_remote_addr：客户端地址的整形格式，对于 IPv4 是 4 字节，对于 IPv6 是 16 字节，所以**在 limit_req 和 limit_conn 中通常可以用作 key** （详见：[Nginx 处理 HTTP 请求的 11 个阶段](https://iziyang.github.io/2020/04/12/5-nginx/) 中的 preaccess 阶段）
- connection：递增的连接序号
- connection_requests：当前连接上执行过的请求数，对 keepalive 连接有意义
- remote_addr：客户端地址
- remote_port：客户端端口
- proxy_protocol_addr：若使用了 proxy_protocol 协议，则返回协议中的地址，否则返回空
- proxy_protocol_port：若使用了 proxy_protocol 协议则返回协议中的端口，否则返回空
- server_addr：服务端地址
- server_port：服务器端端口
- TCP_INFO：TCP 内核层参数，包括 \$tcpinfo_rtt, ​\$tcpinfo_rttvar,​\$tcpinfo_snd_cwnd, $tcpinfo_rcv_space
- server_protocol：服务器端协议，例如 HTTP/1.1

## Nginx 处理请求过程中产生的变量

Nginx 处理 HTTP 请求的过程中也会产生很多变量。

- request_time：请求处理到现在的耗时，单位为秒，精确到毫秒
- server_name：匹配上请求的 server_name 值
- https：如果开启了 TLS/SSL 则返回 on，否则返回空
- request_completion：若请求处理完则返回 OK，否则返回空
- request_id：以 16 进制输出的请求表示 id，该 id 共含有 16 个字节，是随机生成的
- request_filename：待访问文件的完整路径
- document_root：由 URI 和 root、alias 规则生成的文件夹路径
- realpath_root：将 document_root 中的软链接等换成真实路径
- limit_rate：返回客户端响应时的速度上限，单位为每秒字节数。可以通过 set 指令修改对请求产生的效果

## 发送 HTTP 响应时相关的变量

- body_bytes_sent：响应中 body 包体的长度

- bytes_sent：全部 http 响应的长度

- status：http 响应中的返回码

- sent_trailer_名字：把响应结尾内容里的值返回

- sent_http_头部名字：响应中某个具体头部的值

  特殊处理，下面这些变量需要经过特殊处理：

  - sent_http_content_type
  - sent_http_content_length
  - sent_http_location
  - sent_http_last_modified
  - sent_http_connection
  - sent_http_keep_alive
  - sent_http_transfer_encoding
  - sent_http_cache_control
  - sent_http_link

  通用：除了上面这些头部，其他的头部都是通用型的，也就是可以直接拿来用。

## Nginx 系统变量

- time_local：以本地时间标准输出的当前时间，例如 14/Nov/2018:15:55:37 +0800
- time_iso8601：使用 ISO8601 标准输出的当前时间，例如 2018-11-14T15:55:37+08:00
- nginx_version：Nginx 版本号
- pid：所属 worker 进程的进程 id
- pipe：使用了管道则返回 p，否则返回 .
- hostname：所在服务器的主机名，与 hostname 命令输出一致
- msec：1970 年 1 月 1 日到现在的时间，单位为秒，小数点后精确到毫秒

## 实战

配置文件：

```nginx
log_format  vartest  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status bytes_sent=$bytes_sent body_bytes_sent=$body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$sent_http_abc"';

server {
	server_name var.ziyang.com localhost;
	#error_log logs/myerror.log debug;
	access_log logs/vartest.log vartest;
	listen 9090;
	
	location / {
		set $limit_rate 10k;
        # return 200; tcpinfo: $tcpinfo_rtt,$tcpinfo_rttvar, $tcpinfo_snd_cwnd, $tcpinfo_rcv_space 
		return 200 '
arg_a: $arg_a,arg_b: $arg_b,args: $args
connection: $connection,connection_requests: $connection_requests
cookie_a: $cookie_a
uri: $uri,document_uri: $document_uri, request_uri: $request_uri
request: $request
request_id: $request_id
server: $server_addr,$server_name,$server_port,$server_protocol
            
host: $host,server_name: $server_name,http_host: $http_host
limit_rate: $limit_rate
hostname: $hostname
content_length: $content_length
status: $status
body_bytes_sent: $body_bytes_sent,bytes_sent: $bytes_sent
time: $request_time,$msec,$time_iso8601,$time_local
';
	}	
}
```

从上面这个配置文件中，我们可以看出来，返回的响应里面包含了一系列的变量，实际验证一下：

```
➜  test_nginx curl -H 'Content-Length: 0' -H 'Cookie: a=c1' 'localhost:9090?a=1&b=22'

arg_a: 1,arg_b: 22,args: a=1&b=22
connection: 2,connection_requests: 1
cookie_a: c1
uri: /,document_uri: /, request_uri: /?a=1&b=22
request: GET /?a=1&b=22 HTTP/1.1
request_id: 5d40b1ff29d2b87d5db5c4f95ebf5e4d
server: 127.0.0.1,var.ziyang.com,9090,HTTP/1.1
host: localhost,server_name: var.ziyang.com,http_host: localhost:9090
limit_rate: 10240
hostname: yuanzizhen.local
content_length: 0
status: 200
body_bytes_sent: 0,bytes_sent: 0
time: 0.000,1590842354.866,2020-05-30T20:39:14+08:00,30/May/2020:20:39:14 +0800
```

大家可以对比一下响应和配置文件中的值是不是一一对应的，更加深刻的理解一下变量的含义。

好了，这一节咱们学习了。关于 Nginx 的变量就讲完了，下一节讲一下实际应用变量的两个模块，大家会有更深刻的理解。