---
title: python-nginx
date: 2018-05-03 10:28:42
tags: Python
categories: 编程
---

# 几个小问题

## python 处理 nginx 配置文件

最近写一个项目脚本，里面涉及到对 nginx 配置文件的操作，在网上找了几个据说可以操作配置文件的 python 库，但遇到一些问题。

<!-- more -->

- python-nginx

  这个库主要是用来生成配置文件的。

- nginxparser

  这个库可以用来解析配置文件，但是不能通过 pip 安装，否则会无法导入方法，步骤如下：

  ```shell
  # GitHub 地址
  https://github.com/fatiherikli/nginxparser
  # 将项目克隆下来之后，通过 setup.py 文件安装
  python setup.py install
  ```

## 命令行安装 python 库的卸载

stackoverflow链接(https://stackoverflow.com/questions/1550226/python-setup-py-uninstall)

```shell
# 安装
python setup.py install --record files.txt
# 卸载
cat files.txt | xargs rm -rf
```

