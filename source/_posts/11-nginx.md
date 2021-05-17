---
categories: SRE
date: 2020-08-29 12:42:32
title: Nginx 的反向代理流程
tags: Nginx
---

上一篇文章说了 Nginx 的负载均衡，这一篇文章来说一下 Nginx 的反向代理。反向代理和负载均衡通常是分不开的——我们的上游服务器总不会只有一台吧。这篇文章会按照反向代理的流程梳理下每个步骤的配置和指令。

<!-- more -->

## proxy 模块处理请求的流程

先来看第一个反向代理模块 proxy 模块，它接收 HTTP 请求转发给上游服务器的也是 HTTP 请求，下面来看下具体的处理流程，然后再看每一个指令和变量具体是怎么使用的。

### HTTP 反向代理流程

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200620212406)

这幅图里面有一个起点和一个终点，起点就是进入 content 阶段，而终点是关闭或者复用连接。反向代理是通过把请求转发给上游服务，再由上游服务转发过来的。这里面复用连接就是前面讲过的 keepalive 请求。

我们来说一下这幅图的整个流程。

首先请求进来要看一下是否命中了缓存，如果命中了就不需要向上游服务建立连接，直接给用户发送响应即可。如果没有命中缓存，就会继续下面的流程。

下一步就会生成发往上游的 HTTP 头部和包体，注意这一步是在建立连接之前做的，而不是先和上游服务器建立连接。这是因为，Nginx 和上游服务器一般都是在内网，网络环境比较好，而用户侧的网络可能比较慢，所以如果先跟上游服务器建立连接的话，就会占用上游服务器的资源，这个实际上是可以避免的。

然后这个时候其实只收到了用户发来的 Header，如果是 POST 请求的话，还可能会有 body。所以 `proxy_request_buffering` 指令如果打开的话，就会先读取请求的完整包体到缓存中，再与上游服务器建立连接，其实这个指令默认也是打开的。如果是关闭的情况下，就会先跟上游服务器建立连接，然后边读边发送包体，这种情况下，用户侧的网络通常很慢，而上游服务器的并发连接数一般是有限的，但是连接却一直在占用，这是比较消耗上游服务器的资源的，所以通常这个指令都会打开。

根据上面的指令做出判断之后，就会根据负载均衡策略开始选择上游服务器，然后再根据一些参数进行连接，比如 TCP 的一些参数，最后发送请求。

发送完请求之后就开始等待接收响应了，接收完完整的响应头部就开始处理头部，这时候有另外一个指令 `proxy_buffering` 来判断如何发送响应头部，通常这个指令是打开的，因为同样是受限于用户的网络，如果边读边发的话就会占用很长时间，依然是上面提到的问题。

发送完包体之后，就会判断 cache 有没有打开，如果打开了就直接缓存，没打开就进入关闭或者复用连接。

## proxy_pass 指令

下面来看下上面请求处理流程的第一步，就是 proxy_pass 指令。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200621065845)

- 功能：对上游服务使用 HTTP/HTTPS 协议进行反向代理
- 指令：`ngx_http_proxy_module`，默认编译进 Nginx，通过 `--without-http_proxy_module` 禁用

下面是语法规则：

```nginx
Syntax: proxy_pass URL;
Default: —
Context: location, if in location, limit_except
```

指令语法规则比较简单，重点来看一下 URL 参数的规则。

- URL 必须以 http:// 或者 https:// 开头，接下来是域名、IP、Unix Socket 地址或者 upstream 的名字，前两者可以在域名或者 IP 后加端口。最后是可选的 URI
- 当 URL 参数中携带 URI 与否，会导致发向上游请求的 URL 不同：
  - 不携带 URI，则将客户端请求中的 URL 直接转发给上游。location 后使用正则表达式、@ 名字时，应该采用这种方式
  - 携带 URI。则对用户请求中的 URL 作如下操作：将 location 参数中匹配上的一段替换为该 URI
- 该 URI 参数中可以携带变量
- 更复杂的 URL 替换，可以在 location 内的配置添加 rewrite break 语句

### 实战

对于 URL 是否携带参数的两种情况，下面来做一个实际的演练，这是配置文件内容。

```nginx
upstream proxyupstream {
	server 127.0.0.1:8012 weight=1;
}

server {
	server_name proxy.ziyang.com;
    listen 80;
	error_log logs/myerror.log debug;

	location /a {
		proxy_pass http://proxyupstream/www;
		#proxy_method POST;

		proxy_pass_request_headers off;
		# proxy_pass_request_body off;
		proxy_set_body 'hello world!';
		proxy_set_header name '';
		proxy_http_version 1.1;
        proxy_set_header Connection "";
	}
}
```

还记得上一篇文章做过的负载均衡实战吗，不记得的回去看一下，这里需要把里面 8012 这个反向代理的响应改一下，输出 URI 变量。

```nginx
 return 200 '8012 server response.uri:$uri\n';
```

- 当不携带 URI 时

```nginx
proxy_pass http://proxyupstream;
```

```shell
➜  nginx curl proxy.ziyang.com/a/b/c
8012 server response.uri:/a/b/c
```

这时候会发现 URI 是原样输出的。

- 携带 URI

proxy_pass 指令后面加上一个 www 的 URI。

```nginx
proxy_pass http://proxyupstream/www;
```

重载一下配置文件，然后再访问一下，会发现匹配到了 location 的部分被替换成了 proxy_pass 里面的 URI。

```
➜  nginx ./test_nginx/sbin/nginx -s reload
➜  nginx curl proxy.ziyang.com/a/b/c
8012 server response.uri:/www/b/c
```

## 生成发往上游的请求

### 生成请求行

```nginx
Syntax: proxy_method method;
Default: —
Context: http, server, location

Syntax: proxy_http_version 1.0 | 1.1;
Default: proxy_http_version 1.0; 
Context: http, server, location
```

- proxy_method 指令是指定发到上游服务器的方法
- proxy_http_version 是指定与上游服务器建立连接的 HTTP 版本，默认是 1.0，通常我们都会改成 1.1

### 生成请求头部

```nginx
Syntax: proxy_set_header field value;
Default: proxy_set_header Host $proxy_host;
         proxy_set_header Connection close; 
Context: http, server, location

Syntax: proxy_pass_request_headers on | off;
Default: proxy_pass_request_headers on; 
Context: http, server, location
```

头部这里也有两条指令。

- proxy_set_header 这个指令是用来设置头部的，这里需要注意下，如果 value 的值为空，那么这一条 header 就不会向上游发送，相当于没有设置。这条指令也有两个默认值，包括 `Connection close` 头部，所以通常我们都会将 Connection 头部显式的设置成 keepalive
- proxy_pass_request_headers 这一条指令是是否将用户发来的请求头部发给上游服务器，默认是开的

### 生成包体

```nginx
Syntax: proxy_pass_request_body on | off;
Default: proxy_pass_request_body on; 
Context: http, server, location

Syntax: proxy_set_body value;
Default: —
Context: http, server, location
```

- proxy_pass_request_body 和上面类似，表示是否将用户发来的包体转发给上游服务器
- proxy_set_body 这条指令是用来设置 body 内容的

### 实战

还是需要修改一下作为上游服务器的 Nginx 的配置文件，添加上几个变量：

```nginx
server {
    listen 8012;
    default_type text/plain;
    # client_body_in_single_buffer on;
    return 200 '8012 server response.
    uri: $uri
    method: $request_method
    request: $request
    http_name: $http_name
    \n';
}
```

看一下 proxy 里面需要修改的内容，这里面我们先把所有的指令注释掉：

```nginx
location /a {
    proxy_pass http://proxyupstream/www;
    #proxy_method POST;

    # proxy_hide_header aaa;
    # proxy_pass_header server;
    # proxy_ignore_headers X-Accel-Limit-Rate;

    #proxy_pass_request_headers off;
    # proxy_pass_request_body off;
    #proxy_set_body 'hello world!';
    #proxy_set_header name '';
    #proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

这里我们实际请求一下，会看到正常取到了各个头部的值，我添加的 name 头部值也取到了：

```
➜  nginx curl -H 'name: myname' proxy.ziyang.com/a
8012 server response.
        uri: /www
        method: GET
        request: GET /www HTTP/1.0
        http_name: myname
```

然后再改一下配置文件，打开几条指令的注释：

```nginx
proxy_pass http://proxyupstream/www;
proxy_method POST;

# proxy_hide_header aaa;
# proxy_pass_header server;
# proxy_ignore_headers X-Accel-Limit-Rate;

proxy_pass_request_headers off;
# proxy_pass_request_body off;
#proxy_set_body 'hello world!';
#proxy_set_header name '';
proxy_http_version 1.1;
proxy_set_header Connection "";
```

再看一下实际的响应内容：

```nginx
➜  nginx curl -H 'name: myname' proxy.ziyang.com/a
8012 server response.
        uri: /www
        method: POST
        request: POST /www HTTP/1.1
        http_name:
```

我们会发现，method 已经改成了 POST，然后 name 头部的值也取不到了，因为 `proxy_pass_request_headers` 这条指令设置为了 `off`。

构造发往上游服务器的头部会影响上游服务器的处理，包括缓存头部或者 keepalive 等等特性，因此，我们需要能够熟练的掌握如何用 Nginx 来修改请求。

## 接收用户请求包体的方式

下面就该到了接收用户请求的包体了，虽然这是由 Nginx 框架来处理的，但是主要是由反向代理模块来使用的。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200621065302)

前面我们介绍过两种接收用户请求包体的方式，一种是收完再转发，另外一种是边收边发。

```nginx
Syntax: proxy_request_buffering on | off;
Default: proxy_request_buffering on; 
Context: http, server, location
```

那么这两种方式分别适用于什么样的场景呢，就是下面的几种情况，按照自己的实际情况来选择即可。

- 设置为 on
  - 客户端网速较慢
  - 上游服务并发处理能力低
  - 适应高吞吐量场景
- 设置为 off
  - 更及时的响应
  - 降低 Nginx 读写磁盘的消耗
  - 一旦开始发送内容，proxy_next_upstream 功能失败

`proxy_next_upstream ` 这个指令的功能我们后面再介绍。

### 客户端包体的接收

```nginx
Syntax: client_body_buffer_size size;
Default: client_body_buffer_size 8k|16k; 
Context: http, server, location

Syntax: client_body_in_single_buffer on | off;
Default: client_body_in_single_buffer off;
Context: http, server, location
```

- client_body_buffer_size 这条指令主要是用来分配内存的，也有几种情况：
  - 如果接收头部时已经接收完全部的包体，则不分配
  - 如果剩余待接收包体的长度小于 client_body_buffer_size，则仅分配所需大小
  - 分配 client_body_buffer_size 大小内存接收包体，关闭包体缓存时，该内存上的内容及时发送给上游；打开包体缓存时，这一段大小内存用完就会写入临时文件，释放内存，然后再接收下面的包体

- client_body_in_single_buffer 这条指令打开表示请求的 body 我们全部放到内存中，默认是关闭的

### 最大包体长度限制

```nginx
Syntax: client_max_body_size size;
Default: client_max_body_size 1m; 
Context: http, server, location
```

- client_max_body_size 指最大 body 的大小，默认是 1m，如果请求头部中含有 Content-Length 超出指定的大小就会报 413 错误

### 临时文件路径格式

```nginx
Syntax: client_body_temp_path path [level1 [level2 [level3]]];
Default: client_body_temp_path client_body_temp; 
Context: http, server, location

Syntax: client_body_in_file_only on | clean | off;
Default: client_body_in_file_only off; 
Context: http, server, location
```

- client_body_temp_path 是用来设置临时文件的路径的，通过在启动 Nginx 之后 Nginx 就会默认创建临时文件目录，在 Nginx 的根目录下可以看到
- client_body_in_file_only 这个指令是包体必须存放在文件中。这种场景适用于我们需要定位问题的时候，就需要到文件中去排查。
  - on 表示所有上传的 body 会一直保存在文件中
  - clean 表示必须保存到文件中，但是请求处理完成后就会把这个文件删除掉
  - off 表示如果 body 非常小，是不会写入文件的，只有超出了缓存大小，才会写入文件，请求处理完成后也会立即删除

### 读取包体时的超时

```nginx
Syntax: client_body_timeout time;
Default: client_body_timeout 60s; 
Context: http, server, location
```

- client_body_timeout 这条指令指的是接收两条 body 之间的时延，也就是超时时间，如果超出了这个时间就会返回 408 错误

## 向上游服务建立连接

经过上面一系列的处理，以及根据负载均衡算法选择了上游服务器之后，就要开始根据参数来连接上游服务器了。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200621073513)

```nginx
Syntax: proxy_connect_timeout time;
Default: proxy_connect_timeout 60s; 
Context: http, server, location
```

- proxy_connect_timeout 表示在和上游服务器建立 TCP 连接时超时，这时 Nginx 会返回 502，当然如果匹配到的 location 有其他的问题的话，也会直接返回 502，不会等超时时间

```nginx
Syntax: proxy_next_upstream http_502 | ..;
Default: proxy_next_upstream error timeout; 
Context: http, server, location
```

- proxy_next_upstream 这条指令表示当出现一些错误的时候，比如响应码 502，可以再换一台上游服务器，重新去处理

### 上游连接启用 TCP keepalive

```nginx
Syntax: proxy_socket_keepalive on | off;
Default: proxy_socket_keepalive off; 
Context: http, server, location
```

- proxy_socket_keepalive 这里面主要指的是当一条 TCP 连接发送完数据之后，有一个定时器时长，如果超出了这个时长还没有数据发送，就会向服务器发送探测包，用来探测进程是否还存在，如果存在就会发送应答，这个过程是由操作系统来完成的

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200621074230)

### 上游连接启用 HTTP keepalive

```nginx
Syntax: keepalive connections;
Default: —
Context: upstream

Syntax: keepalive_requests number;
Default: keepalive_requests 100; 
Context: upstream
```

这个指令在前面已经说过了，这里再做一个简单的回顾，一个表示开启 keepalive 连接，一个表示保持连接的请求数量。

### 修改 TCP 连接中的 local address

```nginx
Syntax: proxy_bind address [transparent] | off;
Default: —
Context: http, server, location
```

- proxy_bind 这个指令可以指定发给上游服务器的 IP 地址，例如本地可能有多个 IP 地址，有内网和外网的，这时候就可以选择一个 IP 地址发给上游服务器
- 可以使用变量：`proxy_bind $remote_addr`，如果传递不属于本机的 IP 地址时，必须加上参数：`proxy_bind $remote_addr transparent; `

所有的 TCP 连接是基于一个 IP 地址的，实际上就是在改 IP 头中的 Source IP Address：

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200621080540)

### 当客户端关闭连接时

```nginx
Syntax: proxy_ignore_client_abort on | off;
Default: proxy_ignore_client_abort off; 
Context: http, server, location
```

这个指令的意思是说，当客户端关闭连接时，与上游服务器的连接是否要关闭，默认是关闭的，也不建议打开，因为会给上游服务器带来更大的性能压力。

### 向上游发送 HTTP 请求

```nginx
Syntax: proxy_send_timeout time;
Default: proxy_send_timeout 60s; 
Context: http, server, location
```

请求和请求的内容我们在之前的内容中已经说过了，这个指令指的是发送请求的超时时间。

## 接收上游的响应

### 接收上游的 HTTP 响应头部

```nginx
Syntax: proxy_buffer_size size;
Default: proxy_buffer_size 4k|8k; 
Context: http, server, location
```

这个头部表示缓存上游发送回来的 HTTP 响应头部的缓存大小，默认是 4k 或者 8k，如果上游服务器发送的 HTTP 头部特别长，比如有一个很长的 cookie，那么就不能被 Nginx 正常处理了，error.log 中会发现一条日志叫 upstream sent too big header。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200622073301)

在上面反向代理的流程中，是先接收响应头部，然后再处理响应头部的。接收响应头部是由 Nginx 框架 upstream 来接收的，而处理响应头部则是由各种反向代理模块来处理的，例如这里的 proxy 模块。

### 接收上游的 HTTP 包体

下面看一组指令，我会分别介绍这些指令的含义。

```nginx
Syntax: proxy_buffering on | off;
Default: proxy_buffering on; 
Context: http, server, location

Syntax: proxy_buffers number size;
Default: proxy_buffers 8 4k|8k; 
Context: http, server, location

Syntax: proxy_max_temp_file_size size;
Default: proxy_max_temp_file_size 1024m; 
Context: http, server, location

Syntax: proxy_temp_file_write_size size;
Default: proxy_temp_file_write_size 8k|16k; 
Context: http, server, location

Syntax: proxy_temp_path path [level1 [level2 [level3]]];
Default: proxy_temp_path proxy_temp; 
Context: http, server, location
```

- proxy_buffering：通常这个指令是开启的。开启之后会在 Nginx 接收完包体之后再向客户端发送，这样可以使得上游服务器不必一直保持连接，因为通常客户端的网络环境是很差的
  - 这里还有一个头部 X-Accel-Buffering，这个头部是 Nginx 特有的，如果上游服务器发送的响应头部中包含了这个头部，那么就会强制要求 Nginx 接收完包体再发送 
- proxy_buffers：即使开启了 proxy_buffering，也不一定会写入磁盘中，如果包体比较小的话，就会直接写入内存中，大小是由这个指令指定的

下面三个指令是关于临时文件的。

- proxy_max_temp_file_size 表示写入临时文件的最大值，默认是 1G
- proxy_temp_file_write_size 表示每次写入的大小
- proxy_temp_path 表示临时文件存放的目录

### 及时转发包体

```nginx
Syntax: proxy_busy_buffers_size size;
Default: proxy_busy_buffers_size 8k|16k; 
Context: http, server, location
```

- proxy_busy_buffers_size 意思是有些时候我们很可能想要及时的给客户端发送响应，例如当收到一个 1G 的文件的时候，可以先转发 8k 或 16k

### 接收上游时网络速度相关指令

```nginx
Syntax: proxy_read_timeout time;
Default: proxy_read_timeout 60s; 
Context: http, server, location

Syntax: proxy_limit_rate rate;
Default: proxy_limit_rate 0; 
Context: http, server, location
```

- proxy_read_timeout 表示两次读取之间最长超时时间是 60s
- proxy_limit_rate 限制读取上游服务器响应的速度，0 表示不限速

### 上游包体的持久化

```nginx
Syntax: proxy_store_access users:permissions ...;
Default: proxy_store_access user:rw; 
Context: http, server, location

Syntax: proxy_store on | off | string;
Default: proxy_store off; 
Context: http, server, location
```

上游发送的响应可能是一个完整的文件，而我们可能想要存下来然后提供其他服务。

- proxy_store_access 表示存放文件的用户和权限
- proxy_store 表示把临时文件改名存到 root 指令指定的目录

## 处理上游的响应头部

先来回顾一下之前的文章 [Nginx 的过滤模块是干啥用的？](https://iziyang.github.io/2020/05/26/6-nginx/)，这里面我谈到了 Nginx 过滤模块会根据数组中指定的顺序由下到上进行处理。所以响应的头部是会改变 Nginx 的行为的，这里就有一个可以禁用上游响应头部的功能。

### 禁用上游响应头部的功能

```nginx
Syntax: proxy_ignore_headers field ...;
Default: —
Context: http, server, location
```

- 功能：某些响应头部可以改变 Nginx 的行为，使用 proxy_ignore_headers 可以禁用

可以禁用功能的头部：

- X-Accel-Redirect：由上游服务指定在 Nginx 内部重定向，控制请求的执行
- X-Accel-Limit-Rate：由上游设置发往客户端的速度限制，等同 limit_rate 指令
- X-Accel-Buffering：由上游控制是否缓存上游的响应
- X-Accel-Charset：由上游控制 Content-Type 中的 Charset

缓存相关：

- X-Accel-Expires：设置响应在 Nginx 中的缓存时间，单位是秒；@ 开头表示一天内某时刻，是 Nginx 自定义的

- Expires：控制 Nginx 缓存时间，优先级低于 X-Accel-Expires

- Cache-Control：控制 Nginx 缓存时间，优先级低于 X-Accel-Expires

- Set-Cookie：响应中出现 Set-Cookie 则不缓存，可通过 proxy_ignore_headers 禁用

- Vary：响应中出现 Vary 则不缓存，同样可以禁用

### 转发上游的响应：proxy_hide_header 指令

```nginx
Syntax: proxy_hide_header field;
Default: —
Context: http, server, location

Syntax: proxy_pass_header field;
Default: —
Context: http, server, location
```

- 功能：对于上游响应中的某些头部，设置不向客户端转发
- 默认不转发的的响应头部：
  - Date：由 `ngx_http_header_filter_module` 过滤模块填写，值为 Nginx 发送响应头部时的时间，因为这个时间与上游服务器的 Date 值肯定是不一致的
  - Server：同样由 `ngx_http_header_filter_module` 填写，值为 Nginx 版本
  - X-Pad：通常是 Apache 为避免浏览器 BUG 生成的头部，默认忽略
  - X-Accel-：用户控制 Nginx 行为的响应，不需要向客户端转发

除了以上这四类头部之外，如果不想向客户端发送，可以使用 proxy_hide_header 来禁用

- proxy_pass_header：对于已经被 proxy_hide_header 的头部，设置向客户端转发

### 修改返回的 Set-Cookie 头部

```nginx
Syntax: proxy_cookie_domain off;
        proxy_cookie_domain domain replacement;
Default: proxy_cookie_domain off; 
Context: http, server, location

Syntax: proxy_cookie_path off;
        proxy_cookie_path path replacement;
Default: proxy_cookie_path off; 
Context: http, server, location
```

这两个头部都是在修改上游服务器响应头部 Set-Cookie 的内容。

- proxy_cookie_domain：修改域名，可以改成一个新的值
- proxy_cookie_path：可以修改 path 后的值

### 修改返回的 Location 头部

```nginx
Syntax: proxy_redirect default;
        proxy_redirect off;
        proxy_redirect redirect replacement;
Default: proxy_redirect default; 
Context: http, server, location
```

这个指令是对返回的 Location 头部做替换。

经过以上步骤，Nginx 已经处理完了上游的响应头部，接下来就是简单的转发 body 了。

以上我们详细讲了从 proxy_pass 指令处理请求到生成向上游发送的请求的内容再到接收客户端发来的 HTTP body 并与上游服务建立连接并接收上游服务的响应内容以及处理响应头部，以上这些环节就是 Nginx 处理上游服务与客户端之间的所有流程。