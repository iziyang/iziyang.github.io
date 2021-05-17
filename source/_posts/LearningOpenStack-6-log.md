---
title: 【学习】六、OpenStack 日志
date: 2017-09-14 11:25:51
tags: [OpenStack学习,Log,Nova]
categories: SRE
---

## 日志位置

日志一般存放在 `/var/log/...`

## 日志格式

OpenStack 的日志格式都是统一的，如下：



<时间戳><日志等级><代码模块><Request ID><日志内容><源代码位置>

- 时间戳

  日志记录的时间，包括 年 月 日 时 分 秒 毫秒

- 日志等级

  有INFO WARNING ERROR DEBUG等

- 代码模块

  当前运行的模块

- Request ID

  日志会记录连续不同的操作，为了便于区分和增加可读性，每个操作都被分配唯一的Request ID,便于查找日志内容	这是日志的主体，记录当前正在执行的操作和结果等重要信息

- 源代码位置

  日志代码的位置，包括方法名称，源代码文件的目录位置和行号。这一项不是所有日志都有

``` shell
2015-12-10 20:46:49.566 DEBUG nova.virt.libvirt.config [req-5c973fff-e9ba-4317-bfd9-76678cc96584 None None] Generated XML ('<cpu>\n  <arch>x86_64</arch>\n  <model>Westmere</model>\n  <vendor>Intel</vendor>\n  <topology sockets="2" cores="3" threads="1"/>\n  <feature name="avx"/>\n  <feature name="ds"/>\n  <feature name="ht"/>\n  <feature name="hypervisor"/>\n  <feature name="osxsave"/>\n  <feature name="pclmuldq"/>\n  <feature name="rdtscp"/>\n  <feature name="ss"/>\n  <feature name="vme"/>\n  <feature name="xsave"/>\n</cpu>\n',) to_xml /opt/stack/nova/nova/virt/libvirt/config.py:82
```

这条日志我们可以得知：

1. 代码模块是 nova.virt.libvirt.config，由此可知应该是 Hypervisor Libvirt 相关的操作
2. 日志内容是生成 XML
3. 如果要跟踪源代码，可以到 /opt/stack/nova/nova/virt/libvirt/config.py 的 82 行，方法是 to_xml