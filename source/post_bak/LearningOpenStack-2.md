---
title: 【学习】二、Keystone 的一些基本概念
date: 2017-09-07 10:32:15
tags: OpenStack学习
categories: SRE
---

# 基本概念


Keystone的作用：

1. 管理用户及其权限
2. 维护OpenStack的endpoint
3. Authentication（认证）和 Authorization（鉴权）

<!-- more -->

一个用户（user）可以有多个角色（role），user 可以是用户也可以是其他服务，user 可以属于多个 project，project 用于将 OpenStack 的资源进行隔离。

user 访问 OpenStack 时，Keyston 会对其进行验证，它使用的是密码这个时候，然后 Keystone 会给它发一个 Token 作为后续访问的 Credentials。

Credentials 包括：

1. 用户名/密码 
2. Token 
3. API Key
4. 其他高级方式

Token 是由数字和字母组成的字符串，User 成功 Authentication 后 Keystone 生成 Token 并分配给 User。

1. Token 用做访问 Service 的 Credential
2. Service 会通过 Keystone 验证 Token 的有效性
3. Token 的有效期默认是 24 小时

每个service会提供若干个Endpoint，service通过endpoint暴露自己的API。

使用以下命令查看endpoint：

```shell
source devstack/openrc admin admin
openstack catalog list
```

使用以下命令查看角色：

```shell
openstack role list
```

service通过 /etc/nova/policy.json 对 role 进行访问控制。

# Troubleshoot

OpenStack排查问题主要通过日志。

Keystone 主要有两个日志，一个是 keystone.log，一个是 keystone_access.log，在 /var/log/keystone 中。

（在newton版本中，我只看到了第一个日志。）

如果需要得到最详细的日志信息，可以在 /etc/keystone/keystone.conf 中打开 debug 选项。