---
title: 物理机中 Devstack 双节点安装 OpenStck
date: 2017-09-26 11:22:11
tags: [OpenStack,devstack]
categories: SRE
---

# 环境

## 安装系统

最小化安装一个操作系统，这里选择 CentOS 7。



安装 git：

```shell
yum install -y git
```

## 网络

controller：

```shell
# vim /etc/sysconfig/network-scripts/ifcfg-enp4s0

TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp4s0"
UUID="ccf32c18-b4a6-40b8-ae09-06dc4148d42f"
DEVICE="enp4s0"
ONBOOT="yes"
IPADDR="192.168.0.201"
PREFIX="24"
GATEWAY="192.168.0.1"
DNS1="210.41.224.33"
DNS2="192.168.5.1"
IPV6_PRIVACY="no"
```

compute01：

```shell
IPADDR="192.168.0.202"
```

compute02：

```shell
IPADDR="192.168.0.202"
```

# 安装 shake 和 bake

## 添加 DevStack 用户

每个节点都要使用同样的名字。

```shell
useradd -s /bin/bash -d /opt/stack -m stack
```

stack 用户会对系统造成很大的改变，需要无密码的 sudo 权限：

```shell
echo "stack ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

**现在开始使用 stack 用户，退出并以 stack 登录：**

```shell
su - stack
```

## 设置 SSH

在每个节点上设置 stack 用户具有 ssh 权限：

```
mkdir ~/.ssh; chmod 700 ~/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCyYjfgyPazTvGpd8OaAvtU2utL8W6gWC4JdRS1J95GhNNfQd657yO6s1AH5KYQWktcE6FO/xNUC2reEXSGC7ezy+sGO1kj9Limv5vrvNHvF1+wts0Cmyx61D2nQw35/Qz8BvpdJANL7VwP/cFI/p3yhvx2lsnjFE3hN8xRB2LtLUopUSVdBwACOVUmH2G+2BWMJDjVINd2DPqRIA4Zhy09KJ3O1Joabr0XpQL0yt/I9x8BVHdAx6l9U0tMg9dj5+tAjZvMAFfye3PJcYwwsfJoFxC8w/SLtqlFX7Ehw++8RtvomvuipLdmWCy+T9hIkl+gHYE4cS3OIqXH7f49jdJf jesse@spacey.local" > ~/.ssh/authorized_keys
```

## 下载 devstack 脚本

下载最新版本的 devstack：

```shell
git clone https://git.openstack.org/openstack-dev/devstack
cd devstack
```

到目前为止，所有的步骤都适用于所有节点。从现在开始，控制节点和计算节点之间会有一些不同。

## 配置控制节点

控制节点运行 OpenStack 所有的服务，在 devstack 下创建 local.conf：

```shell 
[stack@contorller devstack]$ vim local.conf

[[local|localrc]]
HOST_IP=192.168.0.211
FLAT_INTERFACE=ens33
FIXED_RANGE=10.4.128.0/20
FIXED_NETWORK_SIZE=4096
FLOATING_RANGE=192.168.0.220/29
MULTI_HOST=1
LOGFILE=/opt/stack/logs/stack.sh.log
ADMIN_PASSWORD=root
DATABASE_PASSWORD=root
RABBIT_PASSWORD=root
SERVICE_PASSWORD=root
```

在多节点配置中，私有子网的前十个 IP 通常保留，**运行完 stack.sh 脚本之后**运行：

```shell
for i in `seq 2 10`; do /opt/stack/nova/bin/nova-manage fixed reserve 10.4.128.$i; done
```

开始安装：

```shell
./stack.sh
```

## 配置计算节点

计算节点只运行工作服务，对其他的机器，创建 local.conf：

```shell
[[local|localrc]]
HOST_IP=192.168.0.212 # change this per compute node
FLAT_INTERFACE=ens33
FIXED_RANGE=10.4.128.0/20
FIXED_NETWORK_SIZE=4096
FLOATING_RANGE=192.168.0.220/29
MULTI_HOST=1
LOGFILE=/opt/stack/logs/stack.sh.log
ADMIN_PASSWORD=root
DATABASE_PASSWORD=root
RABBIT_PASSWORD=root
SERVICE_PASSWORD=root
DATABASE_TYPE=mysql
SERVICE_HOST=192.168.0.211
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
ENABLED_SERVICES=n-cpu,q-agt,n-api-meta,c-vol,placement-client
NOVA_VNC_ENABLED=True
NOVNCPROXY_URL="http://$SERVICE_HOST:6080/vnc_auto.html"
VNCSERVER_LISTEN=$HOST_IP
VNCSERVER_PROXYCLIENT_ADDRESS=$VNCSERVER_LISTEN
```

开始安装：

```shell
./stack.sh
```

当每个计算节点都安装完毕之后，验证它的输出：

```shell
nova service-list --binary nova-compute
```

当计算节点的服务出现之后，控制节点上运行：

```shell
./tools/discover_hosts.sh
```

来发现服务。

最终安装失败。

![](http://ww1.sinaimg.cn/large/c552abe7ly1fke5nqi54cj20p602dt8q.jpg)

