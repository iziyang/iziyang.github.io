---
title: 上游服务器出问题了，Nginx 还能获取到响应吗？
categories: SRE
date: 2020-08-29 12:45:16
tags: Nginx
---

在上一篇文章中，我曾提到了一个 `proxy_next_upstream` 指令，这个指令可以在上游服务器出现失败的时候选择其他服务器，这篇文章我们详细来说一下这个功能是怎么使用的。

<!-- more -->

## 上游返回失败时的处理办法

```nginx
Syntax: proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | 
http_504 | http_403 | http_404 | http_429 | non_idempotent | off ...;
Default: proxy_next_upstream error timeout; 
Context: http, server, location
```

当上游服务器出现失败时，可以通过 `proxy_next_upstream` 指令来控制，但是这个指令有一些配置和前提：

- 没有向客户端发送任何内容，只要发送了一个字节，表示上游服务已经生效了
- 参数含义
  - error：在与上游服务建立连接时出现错误，主要指的是网络错误
  - timeout：超时，如 connect timeout、read timeout、write timeout 等
  - invalid_header：收到的上游服务响应头是不合法的
  - http_：可以配置具体的响应状态码来选择一个新的上游服务器
  - non_idempotent：如果客户端是幂等方法（POST，LOCK，PATCH，RFC7231 中定义），配置了这个参数之后，那么是可以继续选择下一个服务器的，默认禁止
  - off：关闭这个指令

## 限制 proxy_next_upstream 的时间与次数

```nginx
Syntax: proxy_next_upstream_timeout time;
Default: proxy_next_upstream_timeout 0; 
Context: http, server, location

Syntax: proxy_next_upstream_tries number;
Default: proxy_next_upstream_tries 0; 
Context: http, server, location
```

- proxy_next_upstream_timeout：设置超时时间，0 表示不限制时间
- proxy_next_upstream_tries：设置重试次数，0 表示不限制次数

### 实战

现在我在本地依然是启动了两个 Nginx，一台作为上游服务器，一台作为反向代理。他们的配置分别如下所示：

- 上游服务 Nginx。这里我监听了 3 个端口一个 8011，一个 8012，一个 8013

```nginx
server {
    listen 8011;
    default_type text/plain;
    #limit_rate 1;
    return 200 '8011 server response.\n';
}
server {
	listen 8012;
	default_type text/plain;
	location / {
		add_header aaa 'aaa value';
	}
    # client_body_in_single_buffer on;
    location /test {
    	return 200 '8012 server response.
            uri: $uri
            method: $request_method
            request: $request
            http_name: $http_name
            \n';
    }
}
server {
    listen 8013;
    default_type text/plain;
    return 500 '8013 server internal error.\n';
}
```

- 反向代理 Nginx。这里我配置了一个 upstream nextups，转发到上游服务 Nginx 的 8013 和 8011 端口

```nginx
upstream nextups {
	server 127.0.0.1:8013;
	server 127.0.0.1:8011;
}

server {
	server_name nextups.ziyang.com;
    listen 80;
	error_log  logs/myerror.log  debug;
	root html/;
	default_type text/plain;
	error_page 500 /a.txt;
	location /error {
		proxy_pass http://nextups;
		proxy_connect_timeout 1s;
		proxy_next_upstream off;
	}
    
    location /a {
	}

	location /intercept {
		proxy_intercept_errors on;
		proxy_pass http://127.0.0.1:8013;
	}
}
```

现在先做一下测试看看会返回什么：

```shell
➜  nginx curl nextups.ziyang.com/error
8013 server internal error.
➜  nginx curl nextups.ziyang.com/error
8011 server response.
```

由于我配置的是 rr 算法，所以一次返回 8013，一次返回 8011。如果继续请求，下一次应该会返回 8013，那来看看当我把监听 8013 的配置取消掉之后看会返回什么：

```shell
➜  nginx curl nextups.ziyang.com/error
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.17.8</center>
</body>
</html>
```

看到没，直接报了 502 错误，因为 8013 端口连接不上了，而且 `proxy_next_upstream off;` 这条指令配置的是 `off` 所以 Nginx 不会去继续请求下一个上游服务器。

现在把 `proxy_next_upstream error;` 配置为 `error` 看会发生什么：

```shell
➜  nginx curl nextups.ziyang.com/error
8011 server response.
➜  nginx curl nextups.ziyang.com/error
8011 server response.
➜  nginx curl nextups.ziyang.com/error
8011 server response.
```

现在就全部返回 8011 了，这说明 `proxy_next_upstream` 指令生效了。

## 用 error_page 拦截上游失败响应

下面这个指令可以配置当上游响应的响应码大于等于 300 时，选择将响应返回客户端还是按 error_page 指令处理，这样，会给用户一个比较好的用户体验。

```nginx
Syntax: proxy_intercept_errors on | off;
Default: proxy_intercept_errors off; 
Context: http, server,
```

- proxy_intercept_errors 配置为 on，那么 error page 就可以生效了

### 实战

依然是上面的配置文件，我配置了 error_page 为 `error_page 500 /a.txt;`，先看下 a.txt 的内容：

```shell
➜  nginx curl nextups.ziyang.com/a.txt
a
```

就是一个简单的字母 a。

然后请求反向代理的 Nginx，由于我上游 Nginx 8013 返回的是 500，所以这时候应该返回这个 error_page，也就是 a：

```shell
➜  nginx curl nextups.ziyang.com/intercept
a
```

的确返回了 a，这是符合我们预期结果的。

这篇文章讲了怎么当上游服务器返回失败或者遇到其他错误时，Nginx 怎么去获取响应，以及当上游服务器故障时如何返回一个用户体验良好的页面。到这里，反向代理和负载均衡的内容就差不多讲完了，下一篇文章开始说一下 Nginx 的缓存。