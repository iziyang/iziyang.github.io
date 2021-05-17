---
title: Nginx 如何自定义变量？
tags: Nginx
categories: SRE
date: 2020-06-14 21:08:19
---

之前的两篇文章 [Nginx 变量介绍](https://iziyang.github.io/2020/06/09/7-nginx/)以及利用 [Nginx 变量做防盗链](https://iziyang.github.io/2020/06/09/8-nginx/) 讲的是 Nginx 有哪些变量以及一个常见的应用。那么如此灵活的 Nginx 怎么能不支持自定义变量呢，今天的文章就来说一下自定义变量的几个模块以及 Nginx 的 keepalive 特性。

<!-- more -->

## 通过映射新变量提供更多的可能性：map 模块

- 功能：基于已有变量，使用类似 switch {case: … default: …} 的语法创建新变量，为其他基于变量值实现功能的模块提供更多的可能性
- 模块：`ngx_http_map_module` 默认编译进 Nginx，通过 `--without-http_map_module` 禁用

### 指令

```nginx
Syntax: map string $variable { ... }
Default: —
Context: http

Syntax: map_hash_bucket_size size;
Default: map_hash_bucket_size 32|64|128; 
Context: http

Syntax: map_hash_max_size size;
Default: map_hash_max_size 2048; 
Context: http
```

我们主要看一下 `map string $variable { ... }` 这个指令。所谓类似 switch case 的语法是指，string 的值可以有多个，可以根据 string 值的不同，来给 $variable 赋不同的值。

### 规则

- 已有变量：string 需要是已有的变量，可以分为下面这三种情况
  - 字符串
  - 一个或者多个变量
  - 变量与字符串的组合
- case 规则：{...} 内的匹配规则需要遵循以下规则，尤其是要注意当使用 hostnames 指令时，与 server name 的匹配规则是一致的，可以看之前的文章 [Nginx 的配置指令](https://iziyang.github.io/2020/04/06/3-nginx/)
  - 字符串严格匹配
  - 使用 hostnames 指令，可以对域名使用前缀 * 泛域名匹配
  - ~ 和 ~* 正则表达式匹配，后者忽略大小写
- default 规则
  - 没有匹配到任何规则时，使用 default
  - 确实 default 时，返回空字符串给新变量
- 其他
  - 使用 include 语法提升可读性
  - 使用 volatile 禁止变量值缓存

大家看到上面这些规则可能都有些晕，废话不多说，直接来看一个实战配置文件就懂了。

### 实战

这里我们有一个配置文件，在这个文件里面我们定义了两个 map 块，分别配置了两个变量，\$name 和 \$mobile，\$name 中包含 hostnames 指令。

```nginx
map $http_host $name {
    hostnames;

    default       0;

    ~map\.ziyang\w+\.org.cn 1;
    *.ziyang.org.cn   2;
    map.ziyang.com   3;
    map.ziyang.*    4;
}

map $http_user_agent $mobile {
    default       0;
    "~Opera Mini" 1;
}

server {
	listen 10001;
	default_type text/plain;
	location /{
		return 200 '$name:$mobile\n';
	}
}
```

下面看一下实际的请求：

```
➜  test_nginx curl -H "Host: map.ziyang.org.cn" 127.0.0.1:10001
2:0
```

为什么会返回 2:0 呢？我们来看一下匹配顺序。

map.ziyang.org.cn 有三个规则可以生效，分别是：

- ~map\.ziyang\w+\.org.cn 1;
- *.ziyang.org.cn   2;
- map.ziyang.*    4;

而泛域名是优先于正则表达式的，* 在前的泛域名优先于在后面的泛域名，因此最终匹配到的就是：

- *.ziyang.org.cn   2;

而第二个变量 $mobile 自然走的是 default 规则，不用多说。

这就是 map 模块的作用，大家可以多尝试一下。

下面再来看一个与 map 模块有点类似的 split_clients 模块，这个模块也是通过生成新的变量来完成 AB 测试功能的，它可以按照变量的值，按照百分比的方式，生成新的变量。

## 实现 AB 测试：split_clients 模块

- 功能：基于已有变量创建新变量，为其他 AB 测试提供更多的可能性
  - 对已有变量的值执行 MurmurHash2 算法，得到 32 位整形哈希数字，记为 hash
  - 32 位无符号整形的最大数字 2^32-1，记为 max
  - 哈希数字与最大数字相除，hash/max，可以得到百分比 percent
  - 配置指令中指示了各个百分比构成的范围，如 0-1%，1%-5% 等，及范围对应的值
  - 当 percent 落在哪个范围里，新变量的值就对应着其后的参数
- 模块：`ngx_http_split_clients_module`，默认编译进 Nginx，通过 `--without-http_split_clients_module` 禁用

### 规则

- 已有变量
  - 字符串
  - 一个或者多个变量
  - 变量与字符串的组合
- case 规则：
  - xx.xx%，支持小数点后 2 位，所有项的百分比相加不能超过 100%
  - *，由它匹配剩余的百分比（100% 减去以上所有项相加的百分比）

### 指令

```nginx
Syntax: split_clients string $variable { ... }
Default: —
Context: http
```

split_clients 的指令与 map 是非常相似的，可以看一下前面的介绍，这里不再赘述了。

下面这个配置，来看下有没有啥问题：

```nginx
split_clients "${http_testcli}" $variant {
    0.51% .one;
    20.0% .two;
    50.5% .three;
    40% .four;
    * "";
}
```

细心的同学可能已经发现了，所有的百分比相加已经超过了 100%，所以 Nginx 直接会抛出一个错误，禁止执行。

```
➜  test_nginx ./sbin/nginx -s reload
nginx: [emerg] percent total is greater than 100% in /Users/mtdp/myproject/nginx/test_nginx/conf/example/17.map.conf:31
```

然后将 `40% .four;` 这一行给屏蔽掉再试试看：

```
➜  test_nginx curl -H "testcli: split_clients.ziyang.com" --resolve "split_clients.ziyang.com:80:127.0.0.1" http://split_clients.ziyang.com
ABtestfile.three
```

正常执行。

## geo 模块

geo 模块与前面两个模块也很相似，不同之处在于，这个模块是基于 IP 地址或者子网掩码这样的变量值来生成新的变量的。

- 功能：根据 IP 地址创建新变量
- 模块：`ngx_http_geo_module`，默认编译进 Nginx，通过 `--without-http_geo_module` 禁用

- 指令

```nginx
Syntax: geo [$address] $variable { ... }
Default: —
Context: http
```

### 规则

- 如果 geo 指令后不输入 $address，那么默认使用 \$remote_addr 变量作为 IP 地址

- {} 内的指令匹配：优先最长匹配

  - 通过 IP 地址及子网掩码的方式，定义 IP 范围，当 IP 地址在范围内时新变量使用其后的参数值

  - default 指定了当以上范围都未匹配上时，新变量的默认值

  - 通过 proxy 指令指定可信地址（参考 realip 模块），此时 remote_addr 的值为 X-Forwarded-For 头部值中最后一个 IP 地址

  - proxy_recursive 允许循环地址搜索

  - include，优化可读性

  - delete 删除指定网络

  ```nginx
geo $country {
      default ZZ;
      #include conf/geo.conf;
      #proxy 172.18.144.211; 
      127.0.0.0/24 US;
      127.0.0.1/32 RU;
      10.1.0.0/16 RU;
      192.168.1.0/24 UK;
  }
  ```

问题：以下命令执行时，变量 country 的值各为多少？（proxy 实际上为客户端地址，这里设置为本机的局域网地址即可，我这里是 172.18.144.211）

```
curl -H 'X-Forwarded-For: 10.1.0.0,127.0.0.2' geo.ziyang.com
curl -H 'X-Forwarded-For: 10.1.0.0,127.0.0.1' geo.ziyang.com
curl -H 'X-Forwarded-For: 10.1.0.0,127.0.0.1,1.2.3.4' geo.ziyang.com
```

结果如下：

```shell
➜  test_nginx curl -H 'X-Forwarded-For: 10.1.0.0,127.0.0.2' geo.ziyang.com
US
➜  test_nginx curl -H 'X-Forwarded-For: 10.1.0.0,127.0.0.1' geo.ziyang.com
RU
➜  test_nginx curl -H 'X-Forwarded-For: 10.1.0.0,127.0.0.1,1.2.3.4' geo.ziyang.com
ZZ
```

这里可以看出来，匹配规则实际上是遵循最长匹配的规则的。

## geoip 模块

geoip 模块可以根据 IP 地址生成对应的地址变量，用法与前面的也都类似，Nginx 是基于 MaxMind 数据库来生成对应的地址的。

- 功能：根据 IP 地址创建新变量
- 模块：`ngx_http_geoip_module`，默认未编译进 Nginx，通过 `--with-http_geoip_module` 禁用

使用这个模块是需要安装 MaxMind 库的，安装步骤如下：

- 安装 MaxMind 里 geoip 的 C 开发库（https://dev.maxmind.com/geoip/legacy/downloadable/ ）
- 编译 Nginx 时带上 `--with-http_geoip_module` 参数
- 下载 MaxMind 中的二进制地址库，这个地址库是需要在指令中指定对应的地址的
- 使用 geoip_country 或者 geoip_city 指令配置好 nginx.conf
- 运行或者升级 Nginx

### geoip_country 指令提供的变量

**指令**

```nginx
Syntax: geoip_country file; # 指定国家类的地址文件
Default: —
Context: http

Syntax: geoip_proxy address | CIDR;
Default: —
Context: http
```

**变量**

- $geoip_country_code：两个字母的国家代码，比如 CN 或者 US
- $geoip_country_code3：三个字母的国家代码，比如 CHN 或者 USA
- $geoip_country_name：国家名称，例如 “China”, “United States”

### geoip_city 指令提供的变量

**指令**

```nginx
Syntax: geoip_city file;
Default: —
Context: http
```

**变量**

- $geoip_latitude：纬度
- $geoip_longitude：经度
- $geoip_city_continent_code：位于全球哪个洲，例如 EU 或 AS
- 与 $geoip_country 指令生成的变量重叠
  - $geoip_country_code：两个字母的国家代码，比如 CN 或者 US
  - $geoip_country_code3：三个字母的国家代码，比如 CHN 或者 USA
  - $geoip_country_name：国家名称，例如 “China”, “United States”
- $geoip_region：洲或者省的编码，例如 02
- $geoip_region_name：洲或者省的名称，例如 Zhejiang 或者 Saint Petersburg
- $geoip_city：城市名
- $geoip_postal_code：邮编号
- $geoip_area_code：仅美国使用的邮编号，例如 408
- $geoip_dma_code：仅美国使用的 DMA 编号，例如 807

## keepalive 模块

前面说的都是 Nginx 的变量相关的内容，其实 Nginx 还有一个很具有特色的模块，那就是 keepalive 模块，由于内容不是很多，所以我就直接写到这篇文章里面了，单写一篇显得内容不够哈。

这里指的是 HTTP 的 keepalive，TCP 也有 keepalive，后面会说。

而且是对客户端的 keepalive，不是对上游服务器的。

- 功能：多个 HTTP 请求通过复用 TCP 连接，可以实现以下功能：

  - 减少握手次数
  - 通过减少并发连接数减少了服务器资源消耗
  - 降低 TCP 拥塞控制的影响，保证滑动窗口维持在一个最优的大小
- Connection 头部

  - close：表示请求处理完就关闭连接
  - keepalive：表示复用连接处理下一条请求
- Keepalive 头部：timeout=n，单位是秒，表示连接至少保持 n 秒

### 指令

对客户端行为控制的指令：

```nginx
Syntax: keepalive_disable none | browser ...;
Default: keepalive_disable msie6; 
Context: http, server, location

Syntax: keepalive_requests number;
Default: keepalive_requests 100; 
Context: http, server, location

Syntax: keepalive_timeout timeout [header_timeout];
Default: keepalive_timeout 75s; 
Context: http, server, location
```

- `keepalive_disable` 设置为 none 表示对所有浏览器启用 keepalive，msie6 表示在老版本 MSIE 上禁用 keepalive
- `keepalive_requests` 设置允许保持 keepalive 的请求的数量
- `keepalive_timeout` 表示超时时间

好了，关于 Nginx 的模块介绍就已经全部介绍完了，有兴趣的同学可以去翻我前面的系列文章。当然还有一部分重要的内容还没有介绍，那就是关于 Nginx 的反向代理和负载均衡部分，这块咱们单独抽出来说，别着急，马上干货就出来。