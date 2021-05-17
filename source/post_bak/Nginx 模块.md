---
title: Nginx 模块
date: 2018-05-10 10:28:42
tags: nginx
categories: code-tech
---

# ngx_http_core_module

<!-- more -->

| absolute_redirect on;                  | 如果禁用，nginx发出的重定向将是相对的。默认开启              |
| -------------------------------------- | ------------------------------------------------------------ |
| aio off;                               | 启用或禁用在FreeBSD和Linux上使用异步文件I / O（AIO）：       |
| aio_write off                          | 如果启用了aio，则指定它是否用于写入文件。目前，这只适用于使用aio线程，并且仅限于使用从代理服务器接收的数据编写临时文件。 |
| alias                                  | 路径别名                                                     |
| chunked_transfer_encoding on           | 允许在HTTP / 1.1中禁用分块传输编码。尽管有标准要求，但使用软件无法支持分块编码时，它可能会派上用场。 |
| client_body_buffer_size 8k\|16k        | 设置读取客户端请求主体的缓冲区大小。                         |
| client_body_in_file_only off           | 确定nginx是否应将整个客户端请求正文保存到文件中。            |
| client_body_in_single_buffer off       | 确定nginx是否应将整个客户端请求主体保存在单个缓冲区中。      |
| client_body_temp_path client_body_temp | 定义一个用于存储持有客户端请求主体的临时文件的目录。指定目录下可以有三层子目录 |
| client_body_timeout 60s                | 定义读取客户端请求主体的超时。                               |
| client_max_body_size 1m                | 设置“Content-Length”请求标题字段中指定的客户端请求主体的最大允许大小。 |
| connection_pool_size 256\|512          | 允许精确调整每个连接的内存分配。                             |
| default_type text/plain                | 定义默认的响应 MIME 类型                                     |
| directio off                           | 允许使用O_DIRECT标志，读取大于或等于指定大小的文件时。       |
| directio_alignment 512                 | 设置directio 队列。                                          |



