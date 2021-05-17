---
title: Nginx 的配置指令
date: 2020-04-06 21:56:09
tags: Nginx
categories: SRE
---

我们已经了解了 Nginx 的基本命令和架构原理，下面该到最让人头疼也是最不容易理解的部分了，那就是 nginx.conf 这个配置文件，下面从 Nginx 的指令开始，一步步来讲解 Nginx 的配置。

<!-- more -->

# Nginx 指令

先来看一个典型的 Nginx 配置文件示例。

```nginx
main
http {
	upstream { … }
	split_clients {…}
	map {…}
	geo {…}
	server {
		if () {…}
		location {
			limit_except {…}
		}
		location {
			location {
			}
		}
	}
	server {
	}
}
```

从上面可以看到，这个配置文件中包含了多个指令块，有些指令块还是重复的，那么这在 Nginx 中是一个什么样的规则？接下来会慢慢介绍。

## 指令块的嵌套

在 Nginx 配置文件中，指令块是可以互相嵌套的，例如上面的示例，http 块中可以包含多个 server 块，server 块中还会包含多个 location 块，每一个块中都有相应的指令。

而每一个指令都有 Context 上下文，也就是生效的环境，这在 Nginx 的官方文档中说的很清楚，例如下面的两条指令，Context 中都表明了各自可以生效的环境，access_log 指令可以在多个上下文中生效：

```nginx
Syntax:  access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
		 access_log off;
Default: access_log logs/access.log combined; 
Context: http, server, location, if in location, limit_except

Syntax:  log_format name [escape=default|json|none] string ...;
Default: log_format combined "..."; 
Context: http
```

## 指令的合并

在 Nginx 中，指令分为两种，一种是值指令，一种是动作类指令：

- 值指令：存储配置项的值，是用来配置某一个配置项的
  - 可以合并
  - 示例
    - root
    - access_log
    - gzip
- 动作类指令：指定行为动作，往往表示接下来要做一件事情
  - 不可以合并
  - 示例
    - rewrit
    - proxy_pass
  - 生效阶段
    - server_rewrite 阶段
    - rewrite 阶段
    - content 阶段

这里面的示例以及生效阶段，后面都还会详细讲，这里可以不用过多关注，既然指令分为两种，那么就有不同的继承规则，下面就来说一下。

## 值指令的继承规则

例如下面的配置文件，这里面在 server 块和 location 块中都配置了 root 指令，Nginx 的继承规则如下：

- 子配置不存在时，直接使用父配置块的指令
- 子配置存在时，覆盖父配置块

```nginx
server {
	listen 8080;
	root /home/geek/nginx/html;
	access_log logs/geek.access.log main;
	location /test {
		root /home/geek/nginx/test;
		access_log logs/access.test.log main;
	}
	location /dlib {
		alias dlib/;
	}
	location / {
	}
```

根据上面这两条规则，第一个 location 使用自家的 root 指令，后面两个 location 则使用 server 块的 root 指令。这和编程语言中变量的作用域也是类似的，作用域更小的变量优先级往往更高，Nginx 的指令也是一样。

## 文档中没有的指令如何判断生效范围

对于很多第三方模块，很可能文档并不完善，这时候需要通过源码来查看指令的生效范围。需要明确下面几个问题：

1. 指令在哪个块下生效？
2. 指令允许出现在哪些块下？

这两个问题是在源码中定义的，例如：

```c
static ngx_command_t  ngx_http_core_commands[] = {

    { ngx_string("variables_hash_max_size"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_MAIN_CONF_OFFSET,
      offsetof(ngx_http_core_main_conf_t, variables_hash_max_size),
      NULL },
    ......
```

从上面第三行可以看到，`variables_hash_max_size` 指令是在 main 块下生效的。

还会有两个回调方法：

- 在 server 块生效，从 http 向 server 合并
  - `char *(*merge_srv_conf)(ngx_conf_t*cf, void *prev, void *conf);`
- 向 location 合并
  - `char *(*merge_loc_conf)(ngx_conf_t*cf, void *prev, void *conf);`

例如：

```c++
static ngx_http_module_t  ngx_http_core_module_ctx = {
    ngx_http_core_preconfiguration,        /* preconfiguration */
    ngx_http_core_postconfiguration,       /* postconfiguration */

    ngx_http_core_create_main_conf,        /* create main configuration */
    ngx_http_core_init_main_conf,          /* init main configuration */

    ngx_http_core_create_srv_conf,         /* create server configuration */
    ngx_http_core_merge_srv_conf,          /* merge server configuration */

    ngx_http_core_create_loc_conf,         /* create location configuration */
    ngx_http_core_merge_loc_conf           /* merge location configuration */
};
```

`ngx_http_module_t` 这个结构体里面，定义了很多回调方法，最后一个 `ngx_http_core_merge_loc_conf` 方法，就是制定合并规则的。这个方法定义了两个参数，一个是父配置，一个是子配置：

```c
static char *ngx_http_core_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_core_loc_conf_t *prev = parent;
    ngx_http_core_loc_conf_t *conf = child;

    ngx_uint_t        i;
    ngx_hash_key_t   *type;
    ngx_hash_init_t   types_hash;

    if (conf->root.data == NULL) {
......
```

这个方法表明了从父配置向子配置合并。

# listen 指令的用法

listen 指令在 server 块中生效，用来配置监听哪些端口，由这些端口来处理请求。listen 指令的配置如下：

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200321111843)

如示例所示，listen 指令可以监听的类型有多种，可以配置监听地址和端口，也可以是仅地址和仅端口，还可以仅监听 IPv6 等等。

# 究竟是哪个 server 来处理请求

## server_name 指令的用法

一个指令：server_name 

server_name 指令是用来配置究竟是哪个 server 来处理我们的请求的。有时候，一个 server_name 中可能会有多个域名，这时候是如何选择的呢？

1. server_name 指令后可以跟多个域名，第一个是主域名，多个域名之间空格分隔
2. 泛域名：仅支持在最前或最后加 *，例如：`server_name *.taohui.tech`
3. 正则表达式匹配：`server_name www.taohui.tech ~^www\d+\.taohui\.tech$;`

当 server_name 指令后有多个域名时，会有一个 server_name_in_redirect 的配置，这个配置默认关闭，它使用来控制域名重定向的，也就是这个配置开启之后，请求过来会重定向到主域名访问。

```nginx
Syntax  server_name_in_redirect on | off;
Default server_name_in_redirect off; 
Context http, server, location
```

4. 还可以用正则表达式创建变量

   - ```nginx
     # 使用 $1/$2 的方式引用变量
     server { 
     	server_name ~^(www\.)?(.+)$; 
     	location / { root /sites/$2; } 
     }
     ```

   - ```nginx
     # 还可以通过加一个 ?<> 的方式来命名变量
     server { 
     	server_name ~^(www\.)?(?<domain>.+)$;
     	location / { root /sites/$domain; } 
     }
     ```

5. 特殊的配置规则

   - .test.tech 可以匹配 test.tech *.test.tech
   - _ 匹配所有域名请求
   - "" 匹配没有传递 host 头部的请求

## server 匹配的顺序

1. 精确匹配（与顺序无关）
2. \* 在前的泛域名（与顺序无关）
3. \* 在后的泛域名（与顺序无关）
4. 按文件中的顺序匹配正则表达式域名
5. default server
   - 第 1 个
   - listen 指定 default

这里面 default server 有两种指定方式，假如没有配置 default server，那么第一个 server 块就会成为 default server，如果 listen 中配置了 default，那么就会由配置的块进行处理。

> 关注公众号回复 Nginx 领取知识图谱

<img src="https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/qrcode_for_gh_3cfae3cf61d9_1280.jpg" style="zoom:15%" />

