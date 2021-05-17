---
title: 【学习】一、虚拟化原理及 OpenStack 架构
date: 2017-09-07 10:15:03
tags: OpenStack学习
categories: SRE
---

# 虚拟化介绍

宿主机通过 Hypervisor 程序将自己的硬件资源虚拟化。

![虚拟化类型](https://xmindshare.s3.amazonaws.com/previews/74HS-h3gsVtg-21096.png)

![1 型虚拟化](http://mmbiz.qpic.cn/mmbiz/Hia4HVYXRicqEj7x6icPfmPjQTsxlM1y8wdazM5KSVTQV3WZbzF1icsbiczJ2FLLiazMY3cehsB7DH6EePMPU5T960uQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

![2 型虚拟化](http://mmbiz.qpic.cn/mmbiz/Hia4HVYXRicqEj7x6icPfmPjQTsxlM1y8wdJia6UK6hicv07e7A657ueQYYfAD9Vr0x9HU26bLibuiatoe4hkkR1YH3dg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

# 虚拟化原理

## **CPU虚拟化**

一个虚拟机在宿主机中其实就是一个qemu-kvm进程，而每一个虚拟的CPU则是进程中的一个线程，CPU可以超配（overcommit）

<!-- more -->

## **内存虚拟化**

kvm需要完成VA虚拟内存-->PA物理内存-->MA机器内存直接的地址转换。虚机OS不能直接访问实际机器内存，所以KVM要负责映射客户物理内存到实际机器内存的转换（PA-->MA）。内存也可以overcommit。

## **存储虚拟化**

KVM的存储虚拟化是通过存储池（Storage Pool）和卷（Volume）来管理的。

Storage Pool 是宿主机上可以看到的一片存储空间，可以是多种类型。

KVM将宿主机目录 /var/lib/libvirt/images 作为默认的 Storge Pool。KVM 是怎么知道要把 /var/lib/libvirt/images 这个目录当做默认 Storage Pool 的呢？KVM 所有可以使用的 Storage Pool 都定义在宿主机的 /etc/libvirt/storage 目录下，每一个 Pool 一个 xml 文件，默认有一个default.xml。

### **KVM 支持多种 Volume 格式**

raw 是默认格式，即原始磁盘镜像格式，移植性好，性能好，但大小固定，不能节省磁盘空间。

qcow2 是推荐使用的格式。cow 表示 copy on write，能够节省磁盘空间，支持 AES 加密，支持 zlib 压缩，支持多快照，功能很多。

vmdk 是 VMWare 的虚拟磁盘格式，也就是说 VMWare 虚机可以直接在 KVM上 运行。这种是目录类型，还有LVM类型的Storge Pool，还有 iscsi，ceph

### **LVM 类型的 Storage Pool**

不仅一个文件可以分配给客户机作为虚拟磁盘，宿主机上 VG 中的 LV 也可以作为虚拟磁盘分配给虚拟机使用。不过，LV 由于没有磁盘的 MBR 引导记录，不能作为虚拟机的启动盘，只能作为数据盘使用。

宿主机上的 VG 就是一个 Storage Pool，VG 中的 LV 就是 Volume。 LV 的优点是有较好的性能；不足的地方是管理和移动性方面不如镜像文件，而且不能通过网络远程使用。

## **网络虚拟化**

虚拟机的虚拟网卡可以通过桥接的方式桥接到物理网卡。

### Linux Bridge

![](http://mmbiz.qpic.cn/mmbiz/Hia4HVYXRicqG0JdAJPQAd62nf3V5USDqo3o0LVFN9DVtjF7JtsJwxl81j9xkOibTCq3h9D3juIDzeFIbO3MTxxYg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

Linux Bridge 是 Linux 上用来做 TCP/IP 二层协议交换的设备，其功能大家可以简单的理解为是一个二层交换机或者 Hub。

> 参考：https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587933&idx=1&sn=958532f257d5b4ba575f297a0b66db25&chksm=8d3081c4ba4708d2199ddfaa151e93da2905eab86e963f194d1a7f90bb77535e5de5b7f83d52&scene=21#wechat_redirect

```shell
esben@all-in-one:~$ brctl show # 查看 Linux Bridge 的配置
bridge name	bridge id		    STP enabled	  interfaces
virbr0		8000.525400d43ff6	yes		      virbr0-nic

virsh list --all # 查看所有的虚拟机
virsh start VM # 启动虚拟机
virsh domiflist VM1 # 查看虚拟机网卡信息
```

> 参考：https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587932&idx=1&sn=d8442e02c9d19114ed2a64b3375b07f6&chksm=8d3081c5ba4708d326b27352349a01c2f2175c0cc7d56944f40af2fc2c64ffdc9a5548f1b89e&scene=21#wechat_redirect

### virbr0

virbr0 是 KVM 默认创建的一个 Bridge，其作用是为连接其上的虚机网卡提供 NAT 访问外网的功能。为连接其上的其他虚拟网卡提供 DHCP 服务。

virbr0 使用 dnsmasq 提供 DHCP 服务，可以在宿主机中查看该进程信息。

```shell
esben@all-in-one:~$ ps aux | grep dnsmasq
esben     6841  0.0  0.0  14224  1024 pts/22   R+   19:14   0:00 grep --color=auto dnsmasq
```

### VLAN

VLAN 将一个交换机分成了多个交换机，限制了广播的范围，在二层将计算机隔离到不同的 VLAN 中。VLAN 的隔离是二层上的隔离，A 和 B 无法相互访问指的是二层广播包（比如 arp）无法跨越 VLAN 的边界。但在三层上（比如IP）是可以通过路由器让 A 和 B 互通的。概念上一定要分清。

通常交换机的端口有两种配置模式： Access 和 Trunk：

![](http://mmbiz.qpic.cn/mmbiz/Hia4HVYXRicqHRxz5AEQC0dXRb158pUFBkOFqeN3sLjCicySVKOxE6Y5PdvlOtcMKVo77SvBBOgEtuAqumBsGgu4g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

#### **Access 口**

这些端口被打上了 VLAN 的标签，表明该端口属于哪个 VLAN。 不同 VLAN 用 VLAN ID 来区分，VLAN ID 的 范围是 1-4096。 Access 口都是直接与计算机网卡相连的。

#### **Trunk 口**

假设有两个交换机 A 和 B，连接 A 和 B 的口就是 Trunk 口，允许所有 VLAN 的数据通过。

![](http://mmbiz.qpic.cn/mmbiz/Hia4HVYXRicqHRxz5AEQC0dXRb158pUFBkPlgx2uhEIl2P4lEVFkrpPWSk6txDV7dKWphjP4KC9COGL2ChLzTmug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

eth0 是宿主机上的物理网卡，有一个命名为 eth0.10 的子设备与之相连。 eth0.10 就是 VLAN 设备了，其 VLAN ID 就是 VLAN 10。

这样的配置其效果就是： 宿主机用软件实现了一个交换机（当然是虚拟的），上面定义了一个 VLAN10。

 eth0.10，brvlan10 和 vnet0 都分别接到 VLAN10 的 Access口上。

而 eth0 就是一个 Trunk 口。VM1 通过 vnet0 发出来的数据包会被打上 VLAN10 的标签。

eth0.10 的作用是：定义了 VLAN10
brvlan10 的作用是：Bridge 上的其他网络设备自动加入到 VLAN10 中

> 参考：https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587920&idx=1&sn=79332fb8fd8370b8d6b7d9728c383008&chksm=8d3081c9ba4708df9fcff17839e3c0fbb53a82298799c15ee45889f830a0087a46672505f115&scene=21#wechat_redirect

#### **Linux Bridge + VLAN = 虚拟交换机**

1. 物理交换机存在多个 VLAN，每个 VLAN 拥有多个端口。 同一 VLAN 端口之间可以交换转发，不同 VLAN 端口之间隔离。 所以交换机其包含两层功能：交换与隔离。
2. Linux 的 VLAN 设备实现的是隔离功能，但没有交换功能。
3. Linux Bridge 专门实现交换功能。
4. Linux Bridge 加 VLAN 在功能层面完整模拟现实世界里的二层交换机。eth0 相当于虚拟交换机上的 trunk 口。

# OpenStack 架构

![openstack_kilo_conceptual_arch](/images/openstack_kilo_conceptual_arch.png)