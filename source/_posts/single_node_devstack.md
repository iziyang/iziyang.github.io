---
title: 虚拟机中 Devstack 单节点安装 OpenStack
date: 2017-09-24 10:44:04
tags: [OpenStack,devstack]
categories: SRE
---

## Ubuntu server 安装及配置

### **安装注意事项**

使用 ubuntu-16.04.3-server-amd64 来进行安装，如果是虚拟化环境，需要启用嵌套虚拟化的选项。



![](http://ww1.sinaimg.cn/large/c552abe7ly1fjuh1hnre7j20aw04awec.jpg)

安装语言最好选择英文，不然会报错，提示说“无法安装busybox-initramfs”。

因为在虚拟机中安装，组件安装要勾选上 Virtual Machine host、OpenSSH server 两个选项。

### **配置**

系统中自带了 vim、screen 等，不必重新安装。

1. 修改 IP 地址

```shell
$ sudo vim /etc/network/interfaces

auto ens33
iface ens33 inet static
address 192.168.0.141
netmask 255.255.255.0
gateway 192.168.0.1
```

2. 修改 DNS 配置

```shell
$ sudo vim /etc/resolvconf/resolv.conf.d/base

nameserver 210.41.224.33
nameserver 192.168.5.1
```

3. 修改安装源

> 此步骤是为了提高安装速度，非必要。

多次安装不成功，均网速的问题。安装的过程，需要考虑从三类位置下载软件：

- OpenStack
- Ubuntu
- Python 

OpenStack：一直没有找到合适稳定的镜像站点。 

Ubuntu：

```shell
$ sudo vim /etc/apt/sources.list

在文件头部添加
deb mirror://mirrors.ubuntu.com/mirrors.txt xenial main restricted universe multiverse
deb mirror://mirrors.ubuntu.com/mirrors.txt xenial-updates main restricted universe multiverse
deb mirror://mirrors.ubuntu.com/mirrors.txt xenial-backports main restricted universe multiverse
deb mirror://mirrors.ubuntu.com/mirrors.txt xenial-security main restricted universe multiverse
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1fjuhgsn4jnj20tg074gm2.jpg)

Python：测试了一下豆瓣 pip 镜像站点速度比较快。 

```shell
$ sudo mkdir /root/.pip/
$ sudo vim /root/.pip/pip.conf

添加如下内容
[global]
index-url = https://pypi.douban.com/simple 
```

4. 执行更新

```shell
$ sudo apt-get update
$ sudo apt-get upgrade
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1fjuhn41d65j20rk09x75f.jpg)

![](http://ww1.sinaimg.cn/large/c552abe7ly1fjuhnhgv14j20hd05ajrg.jpg)

## 安装 OpenStack

### 添加 stack 用户

```shell
$ sudo useradd -s /bin/bash -d /opt/stack -m stack # -s：指定shell -d：指定家目录 -m：创建目录
$ echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
$ sudo su - stack # 切换到 stack 用户
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1fjuhx4p05tj20on03h3yj.jpg)

### 下载 devstack 脚本

```shell
$ sudo pwd
/opt/stack

$ time git clone https://git.openstack.org/openstack-dev/devstack -b stable/ocata
备份脚本
$ tar zcf devstack20170531.tar.gz devstack
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1fjuia6tqmwj20rr09ydgb.jpg)

### 创建配置文件 local.conf

```shell
$ id
$ cd devstack
$ vi local.conf

[[local|localrc]]
ADMIN_PASSWORD=root
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
secret 是初始密码，可以根据需要设置自已的密码。
```

### 开始安装

以 stack 的身份执行以下命令

```shell
$ time ./stack.sh 
```

结果如下：

![](http://ww1.sinaimg.cn/large/c552abe7ly1fjuoc2ejf4j20sl0a40t9.jpg)

