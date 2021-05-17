---
title: 【学习】九、Service Plugin/Agent
date: 2018-01-29 16:17:46
tags: OpenStack学习
categories: SRE
---

Core Plugin/Agent 负责管理核心实体：net, subnet 和 port。而对于更高级的网络服务，则由 Service Plugin/Agent 管理。



Service Plugin 及其 Agent 提供更丰富的扩展功能，包括路由，load balance，firewall等，如图所示：

![](http://mmbiz.qpic.cn/mmbiz/Hia4HVYXRicqHiabflAkZBcRNYx9pp6lLvQicoW9Cfticpriaa0dmXlUrQHwpSsySFpyibFu3TVYMFfuq5NFfX85vJLVQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

**DHCP**

dhcp agent 通过 dnsmasq 为 instance 提供 dhcp 服务。

**Routing**

l3 agent 可以为 project（租户）创建 router，提供 Neutron subnet 之间的路由服务。路由功能默认通过 IPtables 实现。

**Firewall**

L3 agent 可以在 router 上配置防火墙策略，提供网络安全防护。另一个与安全相关的功能是 Security Group，也是通过 IPtables 实现。 Firewall 与 Security Group 的区别在于：

1. Firewall 安全策略位于 router，保护的是某个 project 的所有 network。
2. Security Group 安全策略位于 instance，保护的是单个 instance。

**Load Balance**

Neutron 默认通过 HAProxy 为 project 中的多个 instance 提供 load balance 服务。