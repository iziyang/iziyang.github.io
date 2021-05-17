---
title: 使用分片提高缓存效率
categories: SRE
date: 2020-08-29 12:48:15
tags: Nginx
---

当使用 Nginx 作为反向代理去缓存上游的响应时，如果这个响应特别大，Nginx 去处理一个这么大的响应的时候，效率就比较低下了。特别是当有多个请求并发的去请求一个没有缓存的大文件时，性能存在很大的问题。 Nginx 可以通过一个叫 slice 的模块，来把一个很大的响应分解为小的响应，来提升服务性能。

<!-- more -->

## slice 模块

```nginx
Syntax: slice size;
Default: slice 0; 
Context: http, server, location
```

- 功能：通过 range 协议将大文件分解为多个小文件，更好的用缓存为客户端的 range 协议服务。默认为 0，表示不启用 slice 功能
- 模块：默认没有编译进 `http_slice_module`，通过 `--with-http_slice_module` 启用功能

## slice 模块运行流程

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20200623070941)

## 实战



断点续传、多线程下载