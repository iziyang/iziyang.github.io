---
title: 【学习】七、Neutron 概述
date: 2018-01-29 09:43:49
tags: OpenStack学习
categories: SRE
---

# Neutron 功能

<!-- more -->

> 网络命令 `brctl show`

## 二层交换 Switching

- Neutron 支持多种虚拟交换机，包括 Linux 原生的 Linux Bridge 和 Open vSwitch
- 除了可以创建传统的 VLan 网络，还可以创建基于隧道技术的 Overlay 网络，比如 VxLan 和 GRE（目前 Linux Bridge 只支持 VxLan）
- Open vSwitch 是一个开源的虚拟交换机

## 三层路由 Routing

实例可以配置不同网段的 IP，Neutron 的 router（虚拟路由器）实现跨网段通信。router 通过 IP forwarding、iptables 等技术来实现路由和 NAT。

## 负载均衡 Load Balancing

OpenStack 在 Grizzly 版本第一次引入了 Load-Balancing-as-a-Service（LBaaS），提供了将负载分发到多个实例的能力。LBaaS 支持多种负载均衡产品和方案，不同的实现以 Plugin 的形式集成到 Neutron，目前默认的 Plugin 是 HAProxy。

## 防火墙 Firewalling

Neutron 通过以下两种方式来保障实例和网络的安全性：

- Security Group

  通过 iptables 限制进出实例的网络包

- Firewall-as-a-Service

  FWaas，限制进出虚拟路由器的网络包，也是通过 iptables 实现

# Neutron 网络基本概念

Neutron 管理的网络资源包括 network、subnet 和 port。

## network

network 是一个隔离的二层广播域。Neutron 支持多种类型的 network，包括 local、flat、VLan、VxLan 和 GRE。

- local

  local 网络与其他网络和节点隔离。local 网络中的 instance 只能与位于同一节点上同一网络的 instance 通信，local 网络主要用于单机测试。

- flat

  flat 网络是无 VLan tagging 的网络。flat 网络中的 instance 能与位于同一网络的 instance 通信，并且可以跨多个节点。

- VLan

  VLan 网络是具有 802.1q tagging 的网络。VLan 是一个二层的广播域，同一 VLan 中的 instance 可以通信，不同 VLan 只能通过 router 通信。VLan 网络可跨节点，是应用最广泛的网络类型。

- VxLan

  VxLan 是基于隧道技术的 overlay 网络。VxLan 网络通过唯一的 segmentation ID（也叫 VNI）与其他 VxLan 网络区分。VxLan 中数据包会通过 VNI 封装成 UDP 包进行传输。因为二层的包通过封装在三层传输，能够克服 VLan 和物理网络基础设施的限制。

- GRE

  GRE 是与 VxLan 类似的一种 overlay 网络。主要区别在于使用 IP 包而非 UDP 进行封装。

network 必须属于某一个 Project，Project 中可以创建多个 network。

## subnet

subnet 是一个 IPv4 或者 IPv6 地址段。instance 的 IP 从 subnet 中分配。每个 subnet 需要定义 IP 地址的范围和掩码。

network 与 subnet 是 1对多 关系。

## port

port 可以看做虚拟交换机上的一个端口。port 上定义了 MAC 地址和 IP 地址，当 instance 的虚拟网卡 VIF（Virtual Interface） 绑定到 port 时，port 会将 MAC 和 IP 分配给 VIF。

一个实例可以有多个网卡也即多个端口。

# Neutron 架构

## Neutron Server

对外提供 OpenStack 网络 API，接收请求，并调用 Plugin 处理请求。

## Plugin

处理 Neutron Server 发来的请求，维护 OpenStack 逻辑网络状态， 并调用 Agent 处理请求。

## Agent

处理 Plugin 的请求，负责在 network provider 上真正实现各种网络功能。

## network provider

提供网络服务的虚拟或物理网络设备，例如 Linux Bridge，Open vSwitch 或者其他支持 Neutron 的物理交换机。

## Queue

Neutron Server，Plugin 和 Agent 之间通过 Messaging Queue 通信和调用。

## Database

存放 OpenStack 的网络状态信息，包括 Network, Subnet, Port, Router 等。

以创建一个 VLAN100 的 network 为例，假设 network provider 是 linux bridge， 流程如下：

> 1. **Neutron Server** 接收到创建 network 的请求，通过 **Message Queue**（RabbitMQ）通知已注册的 Linux Bridge **Plugin**。
> 2. **Plugin** 将要创建的 network 的信息（例如名称、VLAN ID等）保存到数据库 **database** 中，并通过 **Message Queue** 通知运行在各节点上的 **Agent**。
> 3. **Agent** 收到消息后会在节点上的物理网卡（比如 eth2）上创建 VLAN 设备（比如 eth2.100），并创建 bridge （比如 brqXXX） 桥接 VLAN 设备。

1. plugin 解决的是 What 的问题，即网络要配置成什么样子？而至于如何配置 How 的工作则交由 agent 完成。
2. plugin，agent 和 network provider 是配套使用的，比如上例中 network provider 是 linux bridge，那么就得使用 linux bridge 的 plungin 和 agent；如果 network provider 换成了 OVS 或者物理交换机，plugin 和 agent 也得替换。
3. plugin 的一个主要的职责是在数据库中维护 Neutron 网络的状态信息，这就造成一个问题：所有 network provider 的 plugin 都要编写一套非常类似的数据库访问代码。为了解决这个问题，Neutron 在 Havana 版本实现了一个 ML2（Modular Layer 2）plugin，对 plugin 的功能进行抽象和封装。有了 ML2 plugin，各种 network provider 无需开发自己的 plugin，只需要针对 ML2 开发相应的 driver 就可以了，工作量和难度都大大减少。
4. plugin 按照功能分为两类： **core plugin** 和 **service plugin**。
   - core plugin 维护 Neutron 的 netowrk, subnet 和 port 相关资源的信息，与 core plugin 对应的 agent 包括 linux bridge, OVS 等
   - service plugin 提供 routing, firewall, load balance 等服务，也有相应的 agent。

