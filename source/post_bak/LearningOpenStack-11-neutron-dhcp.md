---
title: 【学习】十一、Linux Bridge 实现 Neutron 网络
date: 2018-02-03 14:39:08
tags: OpenStack学习
categories: SRE
---

Neutorn ML2 plugin 默认使用的 mechanism driver 是 open vswitch 而不是 linux bridge。但是 linux bridge 技术非常成熟，open vswitch 实现的 Neutron 虚拟网络较为复杂，不易理解；而 linux bridge 方案更直观。先理解 linux bridge 方案后再学习 open vswitch 方案会更容易。

<!-- more -->

# 配置 linux-bridge mechanism driver

配置文件位于` /etc/neutron/neutron.conf`

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFcicEkEnYUqMz6PSmQFAk2mG9xVLySjsr4uRkiaHnuOyl2E8Tiazb8xY6xEEdV76dnHecCibPOQGexzg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

控制节点和计算节点都需要在各自的 neutron.conf 中配置 core_plugin 选项。然后需要让 ML2 使用 linux-bridge mechanism driver。 ML2 的配置文件位于`/etc/neutron/plugins/ml2/ml2_conf.ini` 。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFcicEkEnYUqMz6PSmQFAk2mRKzqVZ1nWJaNwHDllXffiaeyCrpicemS8iaUDJLXibOyHxd5H0OEUeDo9g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

# Local Network

## 在 ML2 中 enable local network

网络示例：

![](http://mmbiz.qpic.cn/mmbiz_jpg/Hia4HVYXRicqHuPEmgHSqDnHEQj1lrpdsZgIkNl8obxcGSakgzkZZYzOBEhI2tYZZzWxhg7icG48AFP2jlGPdltNg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)

local network 的特点是不会与宿主机的任何物理网卡相连，也不关联任何的 VLAN ID。对于每个 local netwrok，ML2 linux-bridge 会创建一个 bridge，instance 的 tap 设备会连接到 bridge。位于同一个 local network 的 instance 会连接到相同的 bridge，这样 instance 之间就可以通信了。

配置文件位于 `/etc/neutron/plugins/ml2/ml2_conf.ini`

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqHuPEmgHSqDnHEQj1lrpdsZwQrxDqkUIw43Uibl3iamwaN1ibZSFdbutpmsvnBDSCujKx2wA1qOUFOtQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

# Flat Network

## 原理与配置

flat network 是不带 tag 的网络，要求宿主机的物理网卡直接与 linux bridge 连接，这意味着：**每个 flat network 都会独占一个物理网卡**。

![](http://mmbiz.qpic.cn/mmbiz_jpg/Hia4HVYXRicqFmELScC0PvNekGZ0UKPibZhcXEDica3EWYD8V0HuU2NuYib4A71SGEyB562ubKmXj4p2MkvXkaBs6cA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)

在 `/etc/neutron/plugins/ml2/ml2_conf.ini` 设置 flat network 相关参数。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFmELScC0PvNekGZ0UKPibZhpiahnMVWSV5ribp47WIEEh6tLh4QZJ1UEUVsKzh0PNEQ7ah3INMLWI7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

需要指明 flat 网络与物理网卡的对应关系：

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFmELScC0PvNekGZ0UKPibZhxdj5icV4WsbR2bjJc8icBt0KPJ0MDnibcibgsljTly0o9AJOkX8rTCn71Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFmELScC0PvNekGZ0UKPibZh9siapcR37hGHRtu7BMpuic7dHZv40ibEO0fTvb5PP9ZnpJ3Ricicvrg6aQg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

- 各个节点中 label 与物理网卡的对应关系可以不一样
- 如果要创建多个 flat 网络，需要定义多个 label，用逗号隔开，也需要用到多个物理网卡

# DHCP 服务的配置

Neutron 提供 DHCP 服务的组件是 DHCP agen，默认通过 dnsmasq 实现 DHCP 功能。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqHQtYZib6TkCAy1krj9g5ZpNloXqDXWYGz3D6xlpsodrO9FFUYUrUQIRQxvszRx0muhyIc4kicMwpcg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

## 配置 DHCP agent

DHCP agent 的配置文件位于`/etc/neutron/dhcp_agent.ini` 。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqHQtYZib6TkCAy1krj9g5ZpNqicnTzmn0mD6H7O2Rd1oWcqe8tSzX5UTSVFjFpPn8HWRiabkgMibXYJyQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

**dhcp_driver**

使用 dnsmasq 实现 DHCP

**interface_driver**

使用 linux bridge 连接 DHCP NameSpace interface

当创建 network 并在 subnet 上 enable DHCP 时，网络节点上的 DHCP agent 会启动一个 dnsmasq 进程为该 network 提供 DHCP 服务。

dnsmasq 是一个提供 DHCP 和 DNS 服务的开源软件。dnsmasq 与 network 是一对一关系，一个 dnsmasq 进程可以为同一 netowrk 中所有 enable 了 DHCP 的 subnet 提供服务。

DHCP agent 会为每个 network 创建一个目录 /opt/stack/data/neutron/dhcp/，用于存放该 network 的 dnsmasq 配置文件。

## dnsmasq 重要的启动参数

**--dhcp-hostsfile**

存放 DHCP host 信息的文件，这里的 host 在我们这里实际上就是 instance。dnsmasq 从该文件获取 host 的 IP 与 MAC 的对应关系。每个 host 对应一个条目，信息来源于 Neutron 数据库。

**--interface**

指定提供 DHCP 服务的 interface。dnsmasq 会在该 interface 上监听 instance 的 DHCP 请求。

## 用 NameSpace 隔离 DHCP 服务

Neutron 通过 dnsmasq 提供 DHCP 服务，而 dnsmasq 如何独立的为每个 network 服务呢？答案是通过 Linux Network NameSpace 隔离。

在二层网络上，VLAN 可以将一个物理交换机分割成几个独立的虚拟交换机。类似地，在三层网络上，Linux network NameSpace 可以将一个物理三层网络分割成几个独立的虚拟三层网络。

每个 NameSpace 都有自己独立的网络栈，包括 route table，firewall rule，network interface device 等。

Neutron 通过 NameSpace 为每个 network 提供独立的 DHCP 和路由服务，从而允许租户创建重叠的网络。如果没有 NameSpace，网络就不能重叠。

**NameSpace 是在 Linux 底层上对进程加以隔离的**。

```
ip netns list  # 列出所有的 namespace
```

其实，宿主机本身也有一个 namespace，叫 root namespace，拥有所有物理和虚拟 interface device。物理 interface 只能位于 root namespace。

新创建的 namespace 默认只有一个 loopback device。管理员可以将虚拟 interface，例如 bridge，tap 等设备添加到某个 namespace。

## veth pair

veth pair 是一种成对出现的特殊网络设备，它们象一根虚拟的网线，可用于连接两个 namespace。向 veth pair 一端输入数据，在另一端就能读到此数据。

可以通过 `ip netns exec <network namespace name> <command>`管理 namespace。例如：

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqHl6KcnHQiashwQKf17p4cK5zpFSiaIoV7Sw9BZZKMFvxrDkdrV0TE7zjN1C6PepTSsZJ8wLibvNq2Ig/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

详见：[获取 DHCP IP 的过程](https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587599&idx=1&sn=d6172afe9207edeebfe1f9c354cf431b&chksm=8d308096ba47098009461d7e0accf2c8212d703d28f0b750f10a64a29cda1e31c908057ddb3f&scene=21#wechat_redirect)

# Vlan Network

## 原理

![](http://mmbiz.qpic.cn/mmbiz_jpg/Hia4HVYXRicqHz8AovDFSFqn216FJawPla38QSl8N5gqdT5ybpvu16O7r5Ft0biaodECqSStHJatbfs4lDaEqUg6A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)

因为物理网卡 eth1 上面可以走多个 vlan 的数据，那么物理交换机上与 eth1 相连的的 port 要设置成 trunk 模式，而不是 access 模式。

## 配置

在 `/etc/neutron/plugins/ml2/ml2_conf.ini` 设置 Vlan network 相关参数。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqGp9viaFtOlXepAF0aGHdC3ggcMqwBGtt09ia6VpJyiakrqkUHXe3mE70AyKQPkrIue5D03TCXJXkokQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqGp9viaFtOlXepAF0aGHdC3gJhJhzA3tpiaAwJWl7X9w6icOTaLG6mosPrCFVjzAZSUU2RnpnzGSpDSw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

上面配置定义了 label 为 “default” 的 vlan network，vlan id 的范围是 3001 - 4000。这个范围是针对普通用户在自己的租户里创建 network 的范围。因为普通用户创建 network 时并不能指定 vlan id，Neutron 会按顺序自动从这个范围中取值。

对于 admin 则没有 vlan id 的限制，admin 可以创建 id 范围为 1-4094 的 vlan network。

接着需要指明 vlan network 与物理网卡的对应关系：

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqGp9viaFtOlXepAF0aGHdC3gRw5w0hOaiaUysr9eww4tq31IAt7IxDBLicQPr8Fc42ibq92JRUWgZBrFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

# Router

## L3 agent

Neutron 的路由服务是由 L3 agent 提供的。 除此之外，L3 agent 通过 iptables 提供 firewall 和 floating ip 服务。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqEoyvBR7B1BUYBcEHA4wiaSkUWvCAqMKEwhszzCo3iaTyBxwiba19tUpoznDKRAnXbyIr4I7I7BXMphg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

## 配置

L3 agent 需要正确配置才能工作，配置文件为 `/etc/neutron/l3_agent.ini`，位于控制节点或网络节点上。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqEoyvBR7B1BUYBcEHA4wiaSkJ5NDZKxrVN9eX8ic9Bfiaib9ayh67TgHv5Scj5Nco9xelFWT3ZdA9IC4Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

interface_driver 是最重要的选项，如果 mechanism driver 是 linux bridge，则：

> interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver

如果选用 open vswitch，则：

> interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver

L3 agent 运行在控制或网络节点上。

```
neutron agent-list
```

l3 agent 会为每个 router 创建了一个 namespace，通过 veth pair 与 TAP 相连，然后将 Gateway IP 配置在位于 namespace 里面的 veth interface 上，这样就能提供路由了。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqEnZYvHvLmHKaicGPf7nmsDfSod9KAT6vcastgaUBeaOXicfliary6yufbcJQCvoyPg7ickR11ZGtfUJA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

## Floating IP

当租户网络连接到 Neutron router，通常将 router 作为默认网关。当 router 接收到 instance 的数据包，并将其转发到外网时:

1. router 会修改包的源地址为自己的外网地址，这样确保数据包转发到外网，并能够从外网返回。
2. router 修改返回的数据包，并转发给真正的 instance。

这个行为被称作 [Source NAT](http://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587316&idx=1&sn=114abfc48e6984309507e523d1548cad&chksm=8d308f6dba47067b5b206331fdbc2391dc508e50697ea26a4012e11763d391dc1f3e1117ff13&scene=21#wechat_redirect)。

如果需要从外网直接访问 instance，则可以利用 floating IP。下面是关于 floating IP 必须知道的事实：

1. floating IP 提供静态 NAT 功能，建立外网 IP 与 instance 租户网络 IP 的一对一映射。
2. floating IP 是配置在 router 提供网关的外网 interface 上的，而非 instance 中。
3. router 会根据通信的方向修改数据包的源或者目的地址。

小结一下：

1. floating IP 能够让外网直接访问租户网络中的 instance。这是通过在 router 上应用 iptalbes 的 NAT 规则实现的。
2. floating IP 是配置在 router 的外网 interface 上的，而非 instance，这一点需要特别注意。

# VXLAN

> 本质上，VXLAN 和 VLAN 都是提供以太网二层服务的。

overlay network 是指建立在其他网络上的网络。overlay network 中的节点可以看作通过虚拟（或逻辑）链路连接起来的。overlay network 在底层可能由若干物理链路组成，但对于节点，不需要关心这些底层实现。目前 linux bridge 只支持 vxlan，不支持 gre；open vswitch 两者都支持。

VXLAN 为 Virtual eXtensible Local Area Network。正如名字所描述的，VXLAN 提供与 VLAN 相同的以太网二层服务，但拥有更强的扩展性和灵活性。

## VXLAN 的优势

1. 支持更多的二层网段

   VLAN 使用 12-bit 标记 VLAN ID，最多支持 4094 个 VLAN，这对大型云部署会成为瓶颈。VXLAN 的 ID （VNI 或者 VNID）则用 24-bit 标记，支持 16777216 个二层网段。

2.  能更好地利用已有的网络路径。

   VLAN 使用 Spanning Tree Protocol 避免环路，这会导致有一半的网络路径被 block 掉。VXLAN 的数据包是封装到 UDP 通过三层传输和转发的，可以使用所有的路径。

3. 避免物理交换机 MAC 表耗尽。

   由于采用隧道机制，TOR (Top on Rack) 交换机无需在 MAC 表中记录虚拟机的信息。

## VXLAN 封装和包格式

VXLAN 是一种在现有物理网络设施中支持大规模多租户网络环境的解决方案。VXLAN 的传输协议是 IP + UDP。

VXLAN 定义了一个 MAC-in-UDP 的封装格式。在原始的 Layer 2 网络包前加上 VXLAN header，然后放到 UDP 和 IP 包中。通过 MAC-in-UDP 封装，VXLAN 能够在 Layer 3 网络上建立起了一条 Layer 2 的隧道。

包格式：

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqEvYIfRUXISJHh3vl6UXnqnaT6MyyVODZlCzKTm3ib2gdaDIJibIMicbOuicz1jZtc3HhgHBfj11iabtLw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

## VXLAN Tunnel EndPoint

VXLAN 使用 VXLAN tunnel endpoint (VTEP) 设备处理 VXLAN 的封装和解封。每个 VTEP 有一个 IP interface，配置了一个 IP 地址。VTEP 使用该 IP 封装 Layer 2 frame，并通过该 IP interface 传输和接收封装后的 VXLAN 数据包。

![](http://mmbiz.qpic.cn/mmbiz_jpg/Hia4HVYXRicqEvYIfRUXISJHh3vl6UXnqn47ZicyJNP8A2TqHkcToLAgCv793Rpe3thcOw3Tg0lGSmcg2E9nXZGVw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)

VXLAN 独立于底层的网络拓扑；反过来，两个 VTEP 之间的底层 IP 网络也独立于 VXLAN。VXLAN 数据包是根据外层的 IP header 路由的，该 header 将两端的 VTEP IP 作为源和目标 IP。

[VXLAN 转发流程](https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587534&idx=1&sn=432430ea6797d72abf02594b3ddcc7ab&chksm=8d308057ba4709419a7ce406d08451ed1abd11341c2624cd61b83d31cdda51e0df35603b3a45&scene=21#wechat_redirect)

## 配置

在 `/etc/neutron/plugins/ml2/ml2_conf.ini` 设置 VXLAN network 相关参数。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqF9SVjNl2BGLFMFqKkan39xExoXR1npfEmcNt5NBZDef75q5HflCUiaicdtX4DaMARdGyDeF3lh3m8w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

然后指定 VXLAN 的范围：

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqF9SVjNl2BGLFMFqKkan39xyeCEM9jgrZ0drSuX8bGPclPSIdiccLtburDQWPJOISP5EUNCkztNeag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

接着需要在 [VXLAN] 中配置 VTEP。

## L2 Population

**L2 Population 是用来提高 VXLAN 网络 Scalability 的**。

当 VXLAN 网络中的主机想要与其他的主机通信时，就会向网络中进行广播，询问对方主机的 MAC 地址，如果网络很大时，广播的成本就很大，所以 L2 Population 就出现了。

L2 Population 的作用是在 VTEP 上提供 Porxy ARP 功能，使得 VTEP 能够预先获知 VXLAN 网络中如下信息：

\1. VM IP -- MAC 对应关系

\2. VM -- VTEP 的对应关系

当 VM A 需要与 VM G 通信时：

\1. Host 1 上的 VTEP 直接响应 VM A 的 APR 请求，告之 VM G 的 MAC 地址。

\2. 因为 Host 1 上的 VTEP 知道 VM G 位于 Host 4，会将封装好的 VXLAN 数据包直接发送给 Host 4 的 VTEP。

这样就解决了 MAC 地址学习和 APR 广播的问题，从而保证了 VXLAN 的 Scalability。

**VTEP 是如何提前获知 IP -- MAC -- VTEP 相关信息的呢？**

答案是：

1. Neutron 知道每一个 port 的状态和信息； port 保存了 IP，MAC 相关数据。
2. instance 启动时，其 port 状态变化过程为：down -> build -> active。
3. 每当 port 状态发生变化时，Neutron 都会通过 RPC 消息通知各节点上的 Neutron agent，使得 VTEP 能够更新 VM 和 port 的相关信息。

VTEP 可以根据这些信息判断出其他 Host 上都有哪些 VM，以及它们的 MAC 地址，这样就能直接与之通信，从而避免了不必要的隧道连接和广播。

### 配置

在 `/etc/neutron/plugins/ml2/ml2_conf.ini` 设置 L2 population mechanism driver。

同时在 [VXLAN] 中配置 enable L2 Population。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFZhpiagtHQnYCd9HaWibQ0gjPfRxiaVL1KIibcmL89R85IibD5u37icGHMpcyjKJoqUbOFYewu9yXKQqjg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

[配置 L2 Population](https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587517&idx=1&sn=cd4d8f849c3ba7a17f37d596aa3976f0&chksm=8d308024ba470932069323b5685b7f2e452904f451a50a225111135135eb9591a0b60fc4d6d4&scene=21#wechat_redirect)

# Securet Group

安全组的原理是通过 iptables 对 instance 所在计算节点的网络流量进行过滤。

## 默认安全组

每个 Project（租户）都有一个命名为 “default” 的默认安全组。

「default」安全组有四条规则，其作用是：**允许所有外出（Egress）的流量，但禁止所有进入（Ingress）的流量**。

```
iptables-save 命令查看相关规则
```

安全组有以下特性：

1. 通过宿主机上 iptables 规则控制进出 instance 的流量。
2. 安全组作用在 instance 的 port 上。
3. 安全组的规则都是 allow，不能定义 deny 的规则。
4. instance 可应用多个安全组叠加使用这些安全组中的规则。

# FWaaS

FWaaS 的原理和传统网络一样，是在 Neutron 虚拟 router 上应用防火墙规则，控制进出租户网络的数据。最根本的实现方式还是 iptables。

FWaaS 有三个重要概念：Firewall、Policy 和 Rule。

**Firewall**

租户能够创建和管理的逻辑防火墙资源。Firewall 必须关联某个 Policy，因此必须先创建 Policy。

**Firewall Policy**

Policy 是 Rule 的集合，Firewall 会按顺序应用 Policy 中的每一条 Rule。

**Firewall Rule**

Rule 是访问控制规则，由源与目的子网 IP、源与目的端口、协议、allow 或 deny 动作组成。

安全组的应用对象是虚拟网卡，由 L2 Agent 实现，比如 neutron_openvswitch_agent 和 neutron_linuxbridge_agent。安全组会在计算节点上通过 iptables 规则来控制进出 instance 虚拟网卡的流量。也就是说：**安全组保护的是 instance**。

FWaaS 的应用对象是 router，可以在安全组之前控制外部过来的流量，但是对于同一个 subnet 内的流量不作限制。也就是说：**FWaaS 保护的是 subnet**

## **启用 FWaaS**

因为 FWaaS 是在 router 中实现的，所以 FWaaS 没有单独的 agent。已有的 L3 agent 负责提供所有 FWaaS 功能。

### 配置 firewall driver

Neutron 在 `/etc/neutron/fwaas_driver.ini` 文件中设置 FWaaS 使用的 driver。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFtmicibc0Vyvsl4oWNECkeWCfyv3ueULLsGopoWfDX63otLH8bYQRC4Foia0ISMU7oP9M4icxZ6YWDEw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

### 配置 Neutron

在 Neutron 配置文件 `/etc/neutron/neutron.conf`  中启用 FWaaS plugin。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqFtmicibc0Vyvsl4oWNECkeWCYCQ0ibJdITYnkkVHEu9Nf5xMWTibXAye6Xuqj2ksgKeHJzxD2wrDq7vw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

无规则的虚拟防火墙，不允许任何流量通过。

FWaaS 用于加强 Neutron 网络的安全性，与安全组可以配合使用。

下面将 FWaaS 和安全组做个比较。

相同点：

1. 底层都是通过 iptables 实现。

不同点：

1. FWaaS 的 iptables 规则应用在 router 上，保护整个租户网络；安全组则应用在虚拟网卡上，保护单个 instance。
2. FWaaS 可以定义 allow 或者 deny 规则；安全组只能定义 allow 规则。

# LBaaS

Load balancer 可以说是分布式系统中比较基础的组件。它接收前端发来的请求，然后将请求按照某种均衡策略转发给后端资源池中的某个处理单元，以完成处理。Load balancer 可以实现系统高可用和横向扩展。

LBaaS 有三个主要的概念：

Pool Member，Pool 和 Virtual IP

**Pool Member**

Pool Member 是 layer 4 的实体，拥有 IP 地址并通过监听端口对外提供服务。

例如 Pool Member 可以是一个 web server，IP 为 172.16.100.9 并通过 80 端口提供 HTTP 服务。

**Pool**

Pool 由一组 Pool Member 组成。这些 Pool Member 通常提供同一类服务。

例如一个 web server pool，包含：

web1：172.16.100.9：80

web2：172.16.100.10：80

**Virtual IP**

Virtual IP 也称作 VIP，是定义在 load balancer 上的 IP 地址。每个 pool member 都有自己的 IP，但对外服务则是通过 VIP。

OpenStack Neutron 目前默认通过 HAProxy 软件来实现 LBaaS。HAProxy 是一个流行的开源 load balancer。Neutron 也支持其他一些第三方 Load balancer。

![](http://mmbiz.qpic.cn/mmbiz_jpg/Hia4HVYXRicqHS5Y0gRQvzvfWibIydbbYTpMHPhiaAdpmCgOK7qZuzVVWVbIsChcDgrBibiaxiboz7YFtGktUtCmjUmMQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)

## 配置 LBaaS agent

配置 LBaaS agent 的地方是 `/etc/neutron/services/loadbalancer/haproxy/lbaas_agent.ini`。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqGujgetPeUicDwic7vmib95M7AZ1JQJEdUx09NRyzUrmas1gicXU7H3O4iaksEBLGmSxRsO51bxmmLX2eA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

interface_driver 的作用是设置 load balancer 的网络接口驱动，可以有两个选项：

Linux Bridge

> interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver

Open vSwitch

> interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver

[配置 LBaaS plugin](https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587495&idx=1&sn=38160c95d3fab28005cb5b7201f7d722&chksm=8d30803eba470928f3b30168e6e3658bc918068553195dd35a421800e3d0c06f8f43373dbc86&scene=21#wechat_redirect)

接下来需要配置：

- 创建 Pool、VIP
- 添加 Pool Member
- 创建 Monitor 来监控负载均衡的状态

## LBaas 实现机制

 Neutron 是如何用 Haproxy 来实现负责均衡的。

在控制节点上运行 ip netns，我们发现 Neutron 创建了新的 namespace qlbaas-xxx

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqE27vtJAmVwPHnjjcmZlCX2jVOaBEaYOEuwFd9oicMAOYQtkTbHFv2h0NWIeZ3kcv6IYg51Q3fm6Xw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

该 namespace 对应我们创建的 pool "web servers"。其命名格式为 qlbaas-< pool ID>。可以通过 ip a 查看其设置。

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqE27vtJAmVwPHnjjcmZlCX2l5YLzx8nDZq83Up3ia0j5Om9F2LwnL656uvXKAGyt5qZKCAU0NI027Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

## Load Balance Method 以及 Session Persistence

1.  Load Balance Method 是为新连接选择 member 的方法
2. Session Persistence 是为同一个 client 的后续连接选择 member 的方法

[两种方法介绍](https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587517&idx=1&sn=cd4d8f849c3ba7a17f37d596aa3976f0&chksm=8d308024ba470932069323b5685b7f2e452904f451a50a225111135135eb9591a0b60fc4d6d4&scene=21#wechat_redirect)





