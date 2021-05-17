---
title: 【学习】四、Nova 综述及 OpenStack 通用设计思路
date: 2017-09-12 19:39:54
tags: [OpenStack学习,Nova]
categories: SRE
---

# nova架构



![nova-f9ad05cc-94a8-408b-90dd-4863227bd681](/images/nova-f9ad05cc-94a8-408b-90dd-4863227bd681.png)

## nova组件

可分为以下几类

### api

**_nova-api_**

接收和响应客户的 API 调用。 除了提供 OpenStack 自己的API，nova-api 还支持 Amazon EC2 API。

<!-- more -->

### compute core

**_nova-scheduler_**

虚机调度服务，负责决定在哪个计算节点上运行虚机

**_nova-compute_**

管理虚机的核心服务，通过调用 Hypervisor API 实现虚机生命周期管理

**_Hypervisor_**

计算节点上跑的虚拟化管理程序，虚机管理最底层的程序。 不同虚拟化技术提供自己的 Hypervisor。 常用的 Hypervisor 有 KVM，Xen， VMWare 等

**_nova-conductor_**

nova-compute 经常需要更新数据库，比如更新虚机的状态。 出于安全性和伸缩性的考虑，nova-compute 并不会直接访问数据库，而是将这个任务委托给 nova-conductor，这个我们后面详细讨论。

### Console Interface

**_nova-console_**

用户可以通过多种方式访问虚机的控制台： nova-novncproxy，基于 Web 浏览器的 VNC 访问 nova-spicehtml5proxy，基于 HTML5 浏览器的 SPICE 访问 nova-xvpnvncproxy，基于 Java 客户端的 VNC 访问

**_nova-consoleauth_**

负责对访问虚机控制台请亲提供 Token 认证

**_nova-cert_**

提供 x509 证书支持

### Database

Nova 会有一些数据需要存放到数据库中，一般使用 MySQL。

数据库安装在控制节点上。 Nova 使用命名为 “nova” 的数据库。

### Message Queue

在前面我们了解到 Nova 包含众多的子服务，这些子服务之间需要相互协调和通信。

为解耦各个子服务，Nova 通过 Message Queue 作为子服务的信息中转站。所以在架构图上我们看到了子服务之间没有直接的连线，是通过 Message Queue 联系的。

# Nova组件如何协同工作

1、只有nova-compute组件需要安装在计算节点上

2、可以使用命令查看nova组件分别安装在哪些节点上

```shell
nova service-list
```

3、虚拟机创建流程：

客户向 api 发送请求，api 传递给消息队列，消息队列通知 scheduler，scheduler 通过调度算法确定在哪一台虚拟机上面创建，发消息给消息队列，消息队列通知该计算节点的 compute 组件，compute 创建虚拟机，发送消息给消息队列，消息队列传递给 conductor 组件，conductor 组件写入更新数据库。

![nova-2](/images/nova-2.png)

# OpenStack通用设计思路

1.  每一个 OpenStack 组件都通过 api 对外提供服务，还可以运行多个 api 实现高可用。

2.  对于某项操作，如果有多个实体能够完成任务，那么一般都会有 scheduler 调度服务，如 nova、cinder。

3.  worker 工作服务：如 nova 的nova-compute服务，如果调度不过来那么可以增加scheduler服务，如果计算资源不够可以增加计算节点。

4.  采用基于 driver 的框架。 以 nova 为例，OpenStack 的计算节点支持多种 Hypervisor。 包括 KVM, Hyper-V, VMWare, Xen, Docker, LXC 等。 nova-compute 为这些 Hypervisor 定义了统一的接口，Hypervisor 只需要实现这些接口，就可以 driver 的形式即插即用到 OpenStack 中。

    ![nova-3](/images/nova-3.png)

    配置文件 /etc/nova/nova.conf 中由 compute_driver 配置项指定该计算节点使用哪种 Hypervisor 的 driver。glance 支持多种存储后端，也是一种 driver 架构，cinder、neutron 都是。

5.  消息队列服务。将每一个服务解耦，实现异步调用。

6.  database。每一个服务的状态信息都在数据库中维护。

