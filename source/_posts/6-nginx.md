---
title: Nginx 的过滤模块是干啥用的？
categories: SRE
date: 2020-05-26 08:46:07
tags: Nginx
---


上一篇文章我写了 [Nginx 的 11 个阶段](https://iziyang.github.io/2020/04/12/5-nginx/)，很多人都说太长了。这是出于文章完整性的考虑的，11 个阶段嘛，一次性说完就完事了。今天这篇文章比较短，看完没问题。

## 过滤模块的位置

之前我们介绍了 Nginx 的 11 个阶段，在 content 阶段时，Nginx 会生成返回给用户的响应内容，对用户的响应内容，实际上还需要做再加工处理，Nginx 的过滤模块就是对响应内容进行再加工处理的。所以实际上过滤模块位于 content 阶段之后，log 阶段之前。

我们先来看一段配置指令：

```nginx
limit_req zone=req_one
burst=120;
limit_conn c_zone 1;

satisfy any;
allow 192.168.1.0/32;
auth_basic_user_file access.pass;

gzip on;
image_filter resize 80 80;
```

那么在这一段配置指令之下，会遵循怎样的请求流程呢？请看一下下面这张图：

<!-- more -->

![](http://ww1.sinaimg.cn/large/c552abe7ly1gf0wjcb6tsj21qx2bc4f8.jpg)

上面这张图的流程大致说一下，如果对于 Nginx 的 11 个阶段不了解的去翻一下之前的文章。

我这里再简单说一下。首先由 Nginx 框架接收 HTTP 请求，经过 preaccess、access、content 阶段的处理，当经过 static 模块之后生成响应的时候，很多时候需要对响应进行处理，然后才会返回给客户端。

这里我们假如响应是一张图片的话，那么需要做缩略图的时候，首先就要经过 image_filter 模块的处理。这里面还有一个 gzip 模块，这两个模块也是需要遵循严格的顺序的。因为如果先做 gzip 压缩的话，缩略图后面就没办法做了。

第二个需要关注的地方是，首先对 header 进行过滤，再对 body 进行过滤。因为我们在对用户发送响应的时候，一定是先发送 header，然后再发送 body，所以所有的过滤模块都会提供对 header 或 body 的过滤，当然 image_filter 和 gzip 模块对这两者都可以过滤。

## 返回响应

前面我们说过，Nginx 的 11 个阶段是有严格的顺序的，而这个顺序是在 Nginx 的代码中以一个数组的形式存在的，这个数组的顺序是从后往前。在给用户返回响应的时候，过滤模块也是有严格顺序的，这个顺序同样是从后往前。来看一下在代码中的定义，标红的是我下面会提到的几个过滤模块。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200522082225)

我们需要重点关注四个过滤模块，它们分别的作用是啥呢？

- copy_filter：复制包体内容

  当我们使用 sendfile 指令的时候，也就是零拷贝技术，不经过用户态内存，这里就是不经过 Nginx 直接发给用户，同时也用了 gzip 模块的时候，gzip 是必须在 copy_filter 模块之后的，因为 gzip 必须对内存中的数据做压缩，这时 copy_filter 就会让 sendfile 指令失效。有些模块不需要对内存中的数据进行处理，就需要在 copy_filter 模块之前进行处理。

- postpone_filter：处理子请求

  用来处理子请求，有些过滤模块需要关心子请求的处理结果，需要放在该模块之后。

- header_filter：构造响应头部

  用来构造最终发送给用户的响应头部，可能会添加一些 Server，Nginx 版本号等内容。

- write_filter：发送响应

  用来实际调用操作系统的 write 或 send 等系统调用，来把响应实际发送出去。

介绍完了过滤模块的功能以及所处的阶段，下面来具体看两个模块。

## sub 模块

介绍一个可以替换响应中字符串内容的模块：sub 模块。

- 功能：将响应中指定的字符串，替换成新的字符串
- 模块：`ngx_http_sub_filter_module` 模块，默认未编译进 Nginx，通过 --with-http_sub_module 启用

### 指令

```nginx
Syntax: sub_filter string replacement;
Default: —
Context: http, server, location

Syntax: sub_filter_last_modified on | off;
Default: sub_filter_last_modified off; 
Context: http, server, location

Syntax: sub_filter_once on | off;
Default: sub_filter_once on; 
Context: http, server, location

Syntax: sub_filter_types mime-type ...;
Default: sub_filter_types text/html; 
Context: http, server, location
```

来解释一下这四个指令都是啥意思。

- `sub_filter string replacement`

  `sub_filter` 指令会把匹配到的 `string` 字符串替换成 `replacement` 表示的字符串。

- `sub_filter_last_modified on | off`

  `sub_filter_last_modified` 指令的意思是，是否要返回原来的 `last_modified` HTTP 头部，因为我们已经修改了文件内容，如果是 `on` 的话，就会继续返回原来的头部。

- `sub_filter_once on | off`

  `sub_filter_once` 的意思是，是否只替换一次，默认打开，如果设置为 `off` 的话，那就会将响应中的内容全部扫描一遍并替换。

- `sub_filter_types mime-type`

  这个指令是说针对那些文件类型进行替换，这里可以设置成 \*，但是效率就会比较低了，需要根据实际情况考虑。

### 实战

配置文件如下：

```nginx
server {
	server_name sub.ziyang.com;
	error_log  logs/myerror.log  info;
	
	location / {
    	sub_filter 'Nginx.oRg'  '$host/nginx';
    	sub_filter 'nginX.cOm' '$host/nginx';
    	#sub_filter_once on;
		sub_filter_once off;
		#sub_filter_last_modified off;
		sub_filter_last_modified on;
	}	
}
```

这里需要重新编译 Nginx，我这里把下一节需要的模块也一起编译进去了：

```shell
./configure --prefix=/Users/mtdp/myproject/nginx/test_nginx --with-http_sub_module --with-http_addition_module --with-http_realip_module
make
cp nginx ../../test_nginx/sbin/ # 复制编译好的 nginx 到之前的目录
```

执行热部署：

> 热部署的流程详见 [Nginx 入门及命令行操作](https://iziyang.github.io/2020/03/10/1-nginx/)

```shell
kill -USR2 87693 # 使用新的 Nginx 二进制文件提供服务
kill -WINCH 87693 # 退出老的 Nginx 的 worker 进程
kill -quit 87693 # 优雅的退出老的 master 进程
```

在浏览器中打开 sub.ziyang.com：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf4ycvl9t8j30ye0ei0ux.jpg)

这里面会发现，nginx.org 已经替换成了 sub.ziyang.com/nginx，nginx.com 也替换成了 sub.ziyang.com/nginx。

## addition 模块

下面再来看一个过滤模块，addition 模块，它可以在响应的前后添加内容。

- 功能：在相应前或者响应后增加内容，增加内容的方式，是通过新增子请求，根据子请求的响应来完成。

- 模块：`ngx_http_addition_filter_module`

  默认未编译进 Nginx，通过 --with-http_addition_module 启用

### 指令

```nginx
Syntax: add_before_body uri;
Default: —
Context: http, server, location

Syntax: add_after_body uri;
Default: —
Context: http, server, location

Syntax: addition_types mime-type ...;
Default: addition_types text/html; 
Context: http, server, location
```

这里的三个指令都比较简单，说一下 `add_before_body` 和 `add_after_body` 后面的 `uri`，这个的意思是说，向指定 `uri` 发起子请求，根据子请求的响应来添加内容。

`addition_types` 指令是指定要添加的文件类型。

### 实战

配置文件如下：

```nginx
server {
	server_name addition.ziyang.com;
	error_log logs/myerror.log info;
	
	location / {
    	#add_before_body /before_action;
    	#add_after_body  /after_action;
		#addition_types *;
	}	
	location /before_action {
		return 200 'new content before\n';
	}
	location /after_action {
		return 200 'new content after\n';
	}
	location /testhost {
		uninitialized_variable_warn on;
		set $foo 'testhost';
		return 200 '$gzip_ratio\n';
	}
}
```

先看在注释掉 addition 模块的指令的情况下，会是什么效果：

```shell
➜  ~ curl addition.ziyang.com/a.txt
a
```

然后打开注释：

```shell
➜  ~ curl addition.ziyang.com/a.txt
new content before
a
new content after
```

在原响应前后都增加了内容。

这里面需要注意一点，在实际情况下，`add_before_body` 和 `add_after_body` 后面是其他的 uri，我这里为了简化，直接转发到对应的 location。

> 本文涉及到的所有配置文件我已经放在了 [Nginx 配置文件](https://docs.qq.com/doc/DSkFoT3pvUVhsaWhT)，大家可以自取。

