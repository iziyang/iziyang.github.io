---
title: 听说你的资源被盗用了，那你知道 Nginx 怎么防盗链吗？
categories: SRE
date: 2020-06-14 14:50:21
tags: Nginx
---

上一篇文章讲了 [Nginx 中的变量和运行原理](https://iziyang.github.io/2020/06/09/7-nginx/)，下面就来说一个主要提供变量并修改变量的值的模块，也就是我们要讲的防盗链模块：referer 模块。

<!-- more -->

## 简单有效的防盗链手段

### 场景

如果做过个人站点的同学，可能会遇到别人盗用自己站点资源链接的情况，这就是盗链。说到盗链就要说一个 HTTP 协议的 头部，referer 头部。当其他网站通过 URL 引用了你的页面，用户在浏览器上点击 URL 时，HTTP 请求的头部会通过 referer 头部将该网站当前页面的 URL 带上，告诉服务器本次请求是由谁发起的。

例如，在谷歌中搜索 Nginx 然后点击链接：

<img src="https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200614144801.229887"  />

在打开的新页面中查看请求头会发现，请求头中包含了 referer 头部且值是 https://www.google.com/。

<img src="https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200614143843.338211"  />

像谷歌这种我们是允许的，但是有一些其他的网站想要引用我们自己网站的资源时，就需要做一些管控了，不然岂不是谁都可以拿到链接。

### 目的

这里目的其实已经很明确了，就是要拒绝非正常的网站访问我们站点的资源。

### 思路

- invalid_referer 变量
  - referer 提供了这个变量，可以用来配置哪些 referer 头部合法，也就是，你允许哪些网站引用你的资源。

## referer 模块

要实现上面的目的，referer 模块可得算头一号，一起看下 referer 模块怎么用的。

- 默认编译进 Nginx，通过 `--without-http_referer_module` 禁用

referer 模块有三个指令，下面看一下。

```nginx
Syntax: valid_referers none | blocked | server_names | string ...;
Default: —
Context: server, location

Syntax: referer_hash_bucket_size size;
Default: referer_hash_bucket_size 64; 
Context: server, location

Syntax: referer_hash_max_size size;
Default: referer_hash_max_size 2048; 
Context: server, location
```

- `valid_referers` 指令，配置是否允许 referer 头部以及允许哪些 referer 访问。
- `referer_hash_bucket_size` 表示这些配置的值是放在哈希表中的，指定哈希表的大小。
- `referer_hash_max_size` 则表示哈希表的最大大小是多大。

这里面最重要的是 `valid_referers` 指令，需要重点来说明一下。

### valid_referers 指令

可以同时携带多个参数，表示多个 referer 头部都生效。

**参数值**

- none
  - 允许缺失 referer 头部的请求访问
- block：允许 referer 头部没有对应的值的请求访问。例如可能经过了反向代理或者防火墙
- server_names：若 referer 中站点域名与 server_name 中本机域名某个匹配，则允许该请求访问
- string：表示域名及 URL 的字符串，对域名可在前缀或者后缀中含有 * 通配符，若 referer 头部的值匹配字符串后，则允许访问
- 正则表达式：若 referer 头部的值匹配上了正则，就允许访问

**invalid_referer 变量**

- 允许访问时变量值为空
- 不允许访问时变量值为 1

### 实战

下面来看一个配置文件。

```nginx
server {
	server_name referer.ziyang.com;
    listen 80;

	error_log logs/myerror.log debug;
	root html;
	location /{
		valid_referers none blocked server_names
               		*.ziyang.com www.ziyang.org.cn/nginx/
               		~\.google\.;
		if ($invalid_referer) {
    			return 403;
		}
		return 200 'valid\n';
	}
}

```

那么对于这个配置文件而言，以下哪些请求会被拒绝呢？

```shell
curl -H 'referer: http://www.ziyang.org.cn/ttt' referer.ziyang.com/
curl -H 'referer: http://www.ziyang.com/ttt' referer.ziyang.com/
curl -H 'referer: ' referer.ziyang.com/
curl referer.ziyang.com/
curl -H 'referer: http://www.ziyang.com' referer.ziyang.com/
curl -H 'referer: http://referer.ziyang.com' referer.ziyang.com/
curl -H 'referer: http://image.baidu.com/search/detail' referer.ziyang.com/
curl -H 'referer: http://image.google.com/search/detail' referer.ziyang.com/
```

我们需要先来解析一下这个配置文件。`valid_referers` 指令配置了哪些值呢？

```nginx
 valid_referers none blocked server_names
        *.ziyang.com www.ziyang.org.cn/nginx/
        ~\.google\.;
```

- none：表示没有 referer 的可以访问
- blocked：表示 referer 没有值的可以访问
- server_names：表示本机 server_name 也就是 referer.ziyang.com 可以访问
- *.ziyang.com：匹配上了正则的可以访问
- www.ziyang.org.cn/nginx/：该页面发起的请求可以访问
- ~\\.google\\.：google 前后都是正则匹配

下面就实际看下响应：

```shell
# 返回 403，没有匹配到任何规则
➜  ~ curl -H 'referer: http://www.ziyang.org.cn/ttt' referer.ziyang.com/
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.17.8</center>
</body>
</html>
➜  ~ curl -H 'referer: http://image.baidu.com/search/detail' referer.ziyang.com/
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.17.8</center>
</body>
</html>
# 匹配到了 *.ziyang.com
➜  ~ curl -H 'referer: http://www.ziyang.com/ttt' referer.ziyang.com/
valid
➜  ~ curl -H 'referer: http://www.ziyang.com' referer.ziyang.com/
valid
# 匹配到了 server name
➜  ~ curl -H 'referer: http://referer.ziyang.com' referer.ziyang.com/
valid
# 匹配到了 blocked
➜  ~ curl -H 'referer: ' referer.ziyang.com/
valid
# 匹配到了 none
➜  ~ curl referer.ziyang.com/
valid
# 匹配到了 ~\.google\.
➜  ~ curl -H 'referer: http://image.google.com/search/detail' referer.ziyang.com/
valid
```

## 防盗链另外一种解决方案：secure_link 模块

referer 模块是一种简单的防盗链手段，必须依赖浏览器发起请求才会有效，如果攻击者伪造 referer 头部的话，这种方式就失效了。

secure_link 模块是另外一种解决的方案。

它的主要原理是，通过验证 URL 中哈希值的方式防盗链。

基本过程是这个样子的：

- 由服务器（可以是 Nginx，也可以是其他 Web 服务器）生成加密的安全链接 URL，返回给客户端
- 客户端使用安全 URL 访问 Nginx，由 Nginx 的 secure_link 变量验证是否通过

原理如下：

- 哈希算法是不可逆的
- 客户端只能拿到执行过哈希算法的 URL
- 仅生成 URL 的服务器，验证 URL 是否安全的 Nginx，这两者才保存原始的字符串
- 原始字符串通常由以下部分有序组成：
  - 资源位置。如 HTTP 中指定资源的 URI，防止攻击者拿到一个安全 URI 后可以访问任意资源
  - 用户信息。如用户的 IP 地址，限制其他用户盗用 URL
  - 时间戳。使安全 URL 及时过期
  - 密钥。仅服务器端拥有，增加攻击者猜测出原始字符串的难度

模块：

- ngx_http_secure_link_module
  - 未编译进 Nginx，需要通过 --with-http_secure_link_module 添加
- 变量
  - secure_link
  - secure_link_expires

```nginx
Syntax: secure_link expression;
Default: —
Context: http, server, location

Syntax: secure_link_md5 expression;
Default: —
Context: http, server, location

Syntax: secure_link_secret word;
Default: —
Context: location
```

### 变量值及带过期时间的配置示例

- secure_link
  - 值为空字符串：验证不通过
  - 值为 0：URL 过期
  - 值为 1：验证通过
- secure_link_expires
  - 时间戳的值

**命令行生成安全链接**

- 生成 md5

```
echo -n '时间戳URL客户端IP密钥' | openssl md5 -binary | openssl base64 | tr +/ - | tr -d =
```

- 构造请求 URL

```
/test1.txt?md5=md5生成值&expires=时间戳（如 2147483647）
```

**Nginx 配置**

- secure_link \$arg_md5,$arg_expires;
  - secure_link 后面必须跟两个值，一个是参数中的 md5，一个是时间戳
- secure_link_md5 \"\$secure_link_expires\$uri\$remote_addr secret\";
  - 按照什么样的顺序构造原始字符串

### 实战

下面是一个实际的配置文件，我这里就不做演示了，感兴趣的可以自己做下实验。

```nginx
server {
	server_name securelink.ziyang.com;
    listen 80;
	error_log  logs/myerror.log  info;
	default_type text/plain;
	location /{
		secure_link $arg_md5,$arg_expires;
        secure_link_md5 "$secure_link_expires$uri$remote_addr secret";

        if ($secure_link = "") {
            return 403;
        }

        if ($secure_link = "0") {
            return 410;
        }

		return 200 '$secure_link:$secure_link_expires\n';
	}

	location /p/ {
        secure_link_secret mysecret2;

        if ($secure_link = "") {
            return 403;
        }

        rewrite ^ /secure/$secure_link;
	}

	location /secure/ {
		alias html/;
    	internal;
	}
}
```

### 仅对 URI 进行哈希的简单办法

除了上面这种相对复杂的方式防盗链，还有一种相对简单的防盗链方式，就是只对 URI 进行哈希，这样当 URI 传

- 将请求 URL 分为三个部分：/prefix/hash/link
- Hash 生成方式：对 “link 密钥” 做 md5 哈希
- 用 `secure_link_secret secret;` 配置密钥

**命令行生成安全链接**

- 原请求
  - link
- 生成的安全请求
  - /prefix/md5/link
- 生成 md5
  - `echo -n 'linksecret' | openssl md5 –hex`

**Nginx 配置**

- `secure_link_secret secret;`

这个防盗链的方法比较简单，那么具体是怎么用呢？大家都在网上下载过资源对吧，不管是电子书还是软件，很多网站你点击下载的时候往往会弹出另外一个页面去下载，这个新的页面其实就是请求的 Nginx 生成的安全 URL。如果这个 URL 被拿到的话，其实还是可以用的，所以需要经常的更新密钥来确保 URL 不会被盗用。

今天这篇文章详细讲了防盗链的具体用法，最近的这两篇文章都是说的已有的变量用法，下一篇文章讲一下怎么生成新的变量。