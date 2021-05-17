---
title: 【学习】十、Neutron 架构总结
date: 2018-01-31 10:21:36
tags: OpenStack学习
categories: SRE
---

# 架构图

<!-- more -->

![](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqF7dQR9rL1uufLL090HfB6Kk8uXjRY9wicfgHlRJDFAyzjBUkgAS1AibbChA2I6gYPcTAAzOhKayGPw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

与 OpenStack 其他服务一样，Neutron 采用的是分布式架构，包括 Neutorn Server、各种 plugin/agent、database 和 message queue。

1. Neutron server 接收 api 请求。
2. plugin/agent 实现请求。
3. database 保存 neutron 网络状态。
4. message queue 实现组件之间通信。

> metadata-agent 
>
> instance 在启动时需要访问 nova-metadata-api 服务获取 metadata 和 userdata，这些 data 是该 instance 的定制化信息，比如 hostname, ip， public key 等。
>
> 但 instance 启动时并没有 ip，那如何通过网络访问到 nova-metadata-api 服务呢？
>
> 答案就是 neutron-metadata-agent。 该 agent 让 instance 能够通过 dhcp-agent 或者 l3-agent 与 nova-metadata-api 通信。

![Neuton 架构总图](http://mmbiz.qpic.cn/mmbiz_jpg/Hia4HVYXRicqF7dQR9rL1uufLL090HfB6KWxMKE7spZS3fqlzSribl0fDZJibAXgo17ziavtMpJPUzOibdjPqMMfyticQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)

- Neutron 通过 plugin 和 agent 提供网络服务
- plugin 位于 Neutron server，包括 core plugin 和 service plugin
- agent 位于各个节点，负责实现网络服务
- core plugin 提供 L2 功能，ML2 是推荐的 plugin
- 使用最广泛的 L2 agent 是 linux bridage 和 open vswitch
- service plugin 和 agent 提供扩展功能，包括 dhcp, routing, load balance, firewall, vpn 等

# 实际网络分类

在实际环境中，通常会有四种网络：

- Management 网络

  用于节点之间消息队列的内部通信和访问数据库服务

- API 网络

  通过 API 网络向用户提供服务，Keystone, Nova, Neutron, Glance, Cinder, Horizon 的 endpoints 均配置在 API 网络上。通常，管理员也通过 API 网络 SSH 管理各个节点

- VM 网络

  用于实例之间的通信，可以选择的类型包括 local, flat, vlan, vxlan 和 gre

- External 网络

  External 网络指的是 VM 网络之外的网络，该网络不由 Neutron 管理。 Neutron 可以将 router attach 到 External 网络，为 instance 提供访问外部网络的能力

