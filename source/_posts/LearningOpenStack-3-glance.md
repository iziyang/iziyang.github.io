---
title: 【学习】三、Glance 以及如何使用 CLI
date: 2017-09-07 10:46:18
tags: [OpenStack学习,Glance]
categories: SRE
---

# Glance架构

![fd94a42d2cb78c98201af77e1d0b7b1e](/images/fd94a42d2cb78c98201af77e1d0b7b1e.png)

## glance-api

glance-api 是系统后台运行的服务进程。 对外提供 REST API，响应 image 查询、获取和存储的调用。

glance-api 不会真正处理请求。 如果操作是与 image metadata（元数据）相关，glance-api 会把请求转发给 glance-registry； 如果操作是与 image 自身存取相关，glance-api 会把请求转发给该 image 的 store backend。



## glance-registry

glance-registry 是系统后台运行的服务进程。 负责处理和存取 image 的 metadata，例如 image 的大小和类型。

**glance支持的镜像格式**

![0ac31c2c324d988dfbfce17f77d16cf9](/images/0ac31c2c324d988dfbfce17f77d16cf9.png)

## Store backend

Glance 自己并不存储 image。 真正的 image 是存放在 backend 中的。 Glance 支持多种 backend，包括

1. A directory on a local file system（这是默认配置）
2. GridFS
3. Ceph RBD
4. Amazon S3
5. Sheepdog
6. OpenStack Block Storage (Cinder)
7. OpenStack Object Storage (Swift)
8. VMware ESX

具体使用哪种 backend，是在 /etc/glance/glance-api.conf中配置的。

```shell
glance image-list # 查看有哪些镜像。
glance image-create --name cirros --file /tmp/cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --progress # 上传镜像，--progress用来显示百分比
glance image-delete imageID
```

# CLI

不同服务用的命令虽然不同，但这些命令使用方式却非常类似，可以举一反三。

## 1. 执行命令之前，需要设置环境变量

这些变量包含用户名、Project、密码等； 如果不设置，每次执行命令都必须设置相关的命令行参数。如：

```shell
$ . admin-openrc
```

## 2. 各个服务的命令都有增、删、改、查的操作

其格式是

```shell
CMD <obj>-create [parm1] [parm2]…
CMD <obj>-delete [parm]
CMD <obj>-update [parm1] [parm2]…
CMD <obj>-list CMD <obj>-show [parm]
```

例如 glance 管理的是 image，那么 CMD 就是 glance，obj 就是 image， 对应的命令就有：

```shell
glance image-create
glance image-delete
glance image-update
glance image-list
glance image-show
```

再比如 neutron 负责管理网络和子网，那么 CMD 就是 neutron，obj 就是 net 和 subnet，对应的命令就有：

**网络相关操作**

```shell
neutron net-create
neutron net-delete
neutron net-update
neutron net-list
neutron net –show
```

**子网相关操作**

``` shell
neutron subnet-create
neutron subnet-delete
neutron subnet-update
neutron subnet-list
neutron subnet–show
```

有的命令 <obj> 可以省略，比如 nova 下面的操作都是针对 instance：

``` shell
nova boot
nova delete
nova list nova show
```

## 3. 每个对象都有 ID

delete，show 等操作都以 ID 为参数

## 4. 使用help查看命令的用法

例如：glance help

查看 glance image-update 的用法：

```shell
glance help image-update
```

# Troubleshooting

日志位于 /etc/log/glance

glance主要有两个日志：glance_api.log 以及 glance_registry.log

- glance-api：记录 REST API 调用情况
- glance-registry：记录 Glance 服务处理请求的过程以及数据库操作

