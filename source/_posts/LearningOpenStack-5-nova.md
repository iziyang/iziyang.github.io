---
title: 【学习】五、Nova 组件详解
date: 2017-09-12 20:21:55
tags: [OpenStack学习,Nova]
categories: SRE
---

# Nova组件详解

## nova-api

nova-api 是 nova 组件的门户，所有对 nova 的请求都首先由 nova-api 处理，nova 对外暴露几个接口，也就是 endpoint，查看 endpoint：

``` shell
openstack endpoint show nova
```

nova-api 对接收到的请求会做如下操作：



1. 检查传递的参数是否合法
2. 调用 nova 的其他子服务来处理请求
3. 格式化其他子服务返回的结果并返回给客户端

**nova-api 可以接受哪些请求？**
简单来说，只要是与虚拟机生命周期管理相关的操作，nova-api 都可以相应，大部分的操作在 dashboard 上面都可以找到，如图所示：

![nova-4](/images/nova-4.png)

## nova-conductor

nova-compute 需要访问数据库获取 instance 的信息，但是 nova-compute 不会直接访问数据库，需要通过 nova-conductor 实现数据的访问。

![nova-5](/images/nova-5.png)

这样做有两个好处：

1. 更好的系统安全性
2. 更好的系统伸缩性

- **更好的系统安全性**

在 OpenStack 的早期版本中，nova-compute 可以直接访问数据库，但这样存在非常大的安全隐患。 因为 nova-compute 这个服务是部署在计算节点上的，为了能够访问控制节点上的数据库，就必须在计算节点的 /etc/nova/nova.conf 中配置访问数据库的连接信息，比如

``` shell
[database]
connection = mysql+pymysql://root:secret@controller/nova?charset=utf8
```

试想任意一个计算节点被黑客入侵，都会导致部署在控制节点上的数据库面临极大风险。为了解决这个问题，从 G 版本开始，Nova 引入了一个新服务 nova-conductor，将 nova-compute 访问数据库的全部操作都放到 nova-conductor 中，而且 nova-conductor 是部署在控制节点上的。 这样就避免了 nova-compute 直接访问数据库，增加了系统的安全性。

- **更好的伸缩性**

nova-conductor 将 nova-compute 与数据库解耦之后还带来另一个好处：提高了 nova 的伸缩性。nova-compute 与 conductor 是通过消息中间件交互的。

这种松散的架构允许配置多个 nova-conductor 实例。 在一个大规模的 OpenStack 部署环境里，管理员可以通过增加 nova-conductor 的数量来应对日益增长的计算节点对数据库的访问。

## nova-scheduler

在 /etc/nova/nova.conf 中，nova 通过 scheduler_driver、scheduler_available_filters、scheduler_default_filters 来配置 nova-scheduler。

### filter_scheduler

filter_scheduler 是 nova-scheduler 默认的调度器，调度过程分为两步：

1. 通过 filter 选择满足条件的计算节点（运行 nova-compute）
2. 通过权重计算（weighting）选择在最优的计算节点上创建 instance

![img](/images/nova-6.png)

调度过程

### Filter

当 filter-scheduler 需要执行调度操作时，会让 filter 对计算节点进行判断，filter 返回 True 或 False。Nova.conf 中的 scheduler_available_filters 选项用于配置 scheduler 可用的 filter，默认是所有 nova 自带的 filter 都可以用于滤操作。

```
scheduler_available_filters = nova.scheduler.filters.all_filters
```

另外还有一个选项 scheduler_default_filters，用于指定 scheduler 真正使用的 filter，默认值如下

```
scheduler_default_filters = RetryFilter, AvailabilityZoneFilter, RamFilter, DiskFilter, ComputeFilter, ComputeCapabilitiesFilter, ImagePropertiesFilter, ServerGroupAntiAffinityFilter, ServerGroupAffinityFilter
```

**RetryFilter**

RetryFilter 的作用是过滤掉已经调度过的节点，比如上次选择部署在计算节点 A 上面，但是由于某种原因失败了，那么这次就不会再部署。

**AvailabilityZoneFilter**

为提高容灾性和提供隔离服务，可以将计算节点划分到不同的Availability Zone中。例如把一个机架上的机器划分在一个 Availability Zone 中。 OpenStack 默认有一个命名为 “Nova” 的 Availability Zone，所有的计算节点初始都是放在 “Nova” 中。 用户可根据需要创建自己的 Availability Zone。nova-scheduler 会将不属于指定 AvailableZone 的计算节点过滤掉。

**RamFilter**

RamFilter 将不能满足 flavor 内存需求的计算节点过滤掉。

对于内存有一点需要注意： 为了提高系统的资源使用率，OpenStack 在计算节点可用内存时允许 overcommit，也就是可以超过实际内存大小。 超过的程度是通过 nova.conf 中 ram_allocation_ratio 这个参数来控制的，默认值为 1.5

```
ram_allocation_ratio = 1.5
```

其含义是：如果计算节点的内容为 10GB，OpenStack 则会认为它有 15GB（10*1.5）内存。

**DiskFilter**

DiskFilter 将不能满足 flavor 磁盘需求的计算节点过滤掉。Disk 同样允许 overcommit，通过 nova.conf 中 disk_allocation_ratio 控制，默认值为 1

```
disk_allocation_ratio = 1.0
```

**CoreFilter**

CoreFilter 将不能满足 flavor vCPU 需求的计算节点过滤掉。vCPU 同样允许 overcommit，通过 nova.conf 中 cpu_allocation_ratio 控制，默认值为 16

```
cpu_allocation_ratio = 16.0
```


**ComputeFilter**

ComputeFilter 保证只有 nova-compute 服务正常工作的计算节点才能够被 nova-scheduler调度。ComputeFilter 显然是必选的 filter。

**ComputeCapabilitiesFilter**

ComputeCapabilitiesFilter 根据计算节点的特性来筛选。这个比较高级，我们举例说明。
例如我们的节点有 x86_64 和 ARM 架构的，如果想将 Instance 指定部署到 x86_64 架构的节点上，就可以利用 ComputeCapabilitiesFilter。还记得 flavor 中有个 Metadata 吗，Compute 的 Capabilitie s就在 Metadata中 指定。

**ImagePropertiesFilter**

ImagePropertiesFilter 根据所选 image 的属性来筛选匹配的计算节点。 跟 flavor 类似，image 也有 metadata，用于指定其属性。

例如希望某个 image 只能运行在 kvm 的 hypervisor 上，可以通过 “Hypervisor Type” 属性来指定。

![](http://mmbiz.qpic.cn/mmbiz/Hia4HVYXRicqHfDp3xocW9A8pmGO7lSjGEruoobfuj7ib4puicFr8UiaAWVYzf3WmI4PicrOPOWxoM5XQYqLiboEajvoA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

**ServerGroupAntiAffinityFilter**

ServerGroupAntiAffinityFilter 可以尽量将 Instance 分散部署到不同的节点上。

例如有 inst1，inst2 和 inst3 三个 instance，计算节点有 A,B 和 C。 为保证分散部署，进行如下操作：

1. 创建一个 anti-affinity 策略的 server group “group-1”

```
nova server-group-create --policy anti-affinity group-1
```

请注意，这里的 server group 其实是 instance group，并不是计算节点的 group。

2. 依次创建 Instance inst1, inst2和inst3并放到group-1中

```
nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst1
nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst2
nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst3
```

因为 group-1 的策略是 AntiAffinity，调度时 ServerGroupAntiAffinityFilter 会将 inst1, inst2 和 inst3 部署到不同计算节点 A, B 和 C。目前只能在 CLI 中指定 server group 来创建 instance。

创建 instance 时如果没有指定 server group，ServerGroupAntiAffinityFilter 会直接通过，不做任何过滤。

**ServerGroupAffinityFilter**

与 ServerGroupAntiAffinityFilter 的作用相反，ServerGroupAffinityFilter 会尽量将 instance 部署到同一个计算节点上。 方法类似

1. 创建一个 affinity 策略的 server group “group-2”

```
nova server-group-create --policy affinity group-2
```

2. 依次创建 instance inst1, inst2 和 inst3 并放到 group-2 中

```
nova boot --image IMAGE_ID --flavor 1 --hint group=group-2 inst1
nova boot --image IMAGE_ID --flavor 1 --hint group=group-2 inst2
nova boot --image IMAGE_ID --flavor 1 --hint group=group-2 inst3
```

因为 group-2 的策略是 Affinity，调度时 ServerGroupAffinityFilter 会将 inst1, inst2 和 inst3 部署到同一个计算节点。创建 instance 时如果没有指定 server group，ServerGroupAffinityFilter 会直接通过，不做任何过滤。

### weight

通过过滤之后会计算权重值，根据计算节点空闲内存来计算权重值，内存越多，权重值越大。

### 日志

```
 /var/log/nova/scheduler.log
```

注：要显示 DEBUG 日志，需要在 /etc/nova/nova.conf 中打开 debug 选项

```
[DEFAULT]
debug = True
```

## nova-compute

nova-compute 通过 Driver架构支持多种 Hypervisor。

![](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HOIic9qUBjiawxfMhd9Uu6iaHA/0?wx_lazy=1)

我们可以在 /opt/stack/nova/nova/virt/ 目录下查看到 OpenStack 源代码中已经自带了上面这几个 Hypervisor 的 Driver：

![](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HEvkgnaPF0j6JpZhkhic3FSA/0?wx_lazy=1)

某个特定的计算节点上只会运行一种 Hypervisor，只需在该节点 nova-compute 的配置文件 /etc/nova/nova.conf 中配置所对应的 compute_driver 就可以了。

![](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HVQzKbULzbJ36Lk63COsOgw/0?wx_lazy=1)

nova-compute 的功能可以分为两类：

1. 定时向 OpenStack 报告计算节点的状态
2. 实现 instance 生命周期的管理

### 定时向 OpenStack 报告计算节点的状态

前面我们看到 nova-scheduler 的很多 Filter 是根据算节点的资源使用情况进行过滤的。比如 RamFilter 要检查计算节点当前可以的内存量；CoreFilter 检查可用的 vCPU 数量；DiskFilter 则会检查可用的磁盘空间。

那这里有个问题：OpenStack 是如何得知每个计算节点的这些信息呢？答案是 nova-compute 会定期向 OpenStack 报告。

nova-compute 是如何获得当前计算节点的资源使用信息的？nova-compute 可以通过 Hypervisor 的 driver 拿到这些信息。

### 实现 instance 生命周期的管理

OpenStack 对 instance 最主要的操作都是通过 nova-compute 实现的，包括 instance 的 launch、shutdown、reboot、suspend、resume、terminate、resize、migration、snapshot 等。

*本节讨论实例的创建操作。*

nova-compute 创建 instance 的过程可以分为 4 步：

1. 为 instance 准备资源
2. 创建 instance 的镜像文件
3. 创建 instance 的 XML 定义文件
4. 创建虚拟网络并启动虚拟机

#### 为 instance 准备资源

nova-compute 首先会根据指定的 flavor 依次为 instance 分配内存、磁盘空间和 vCPU。可以在日志中看到这些细节。

![](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HyvKUuR3MJKTrRtEiaqCUuGA/0?wx_lazy=1)

网络资源也会提前分配。

![](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HM75tW5vXVlN2bazicRvd1lA/0?wx_lazy=1)

#### 创建 instance 的镜像文件

OpenStack 启动一个 instance 时，会选择一个 image，这个 image 由 Glance 管理。 nova-compute会：

1. 首先将该 image 下载到计算节点
2. 然后将其作为 backing file 创建 instance 的镜像文件

**从 Glance 下载 image**

nova-compute 首先会检查 image 是否已经下载（比如之前已经创建过基于相同 image 的 instance）。如果没有，就从 Glance 下载 image 到本地。由此可知，如果计算节点上要运行多个相同 image 的 instance，只会在启动第一个 instance 的时候从 Glance 下载 image，后面的 instance 启动速度就大大加快了。

![](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HNAxlaVXZswZhEfUKMfHlYg/0?wx_lazy=1)

可以看到：

1. image（ID为 917d60ef-f663-4e2d-b85b-e4511bb56bc2）是 qcow2 格式，nova-compute 将其下载。Nova 默认会通过 qemu-img 转换成 raw 格式，以提高 IO 性能。
2. image 的存放目录是 `/opt/stack/data/nova/instances/_base`，这是由 `/etc/nova/nova.conf` 的下面两个配置选项决定的。 

```
instances_path = /opt/stack/data/nova/instances
base_dir_name = _base
```

3. 下载的 image 文件被命名为 60bba5916c6c90ed2ef7d3263de8f653111dd35f，这是 image id 的 SHA1 哈希值。

#### 为 instance 创建镜像文件

有了 image 之后，instance 的镜像文件直接通过 qemu-img 命令创建，backing file 就是下载的 image。

![http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20160501-1462109263590069817.png](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HTq97sJz331iboFhVFpcVgIg/0?wx_lazy=1)

可以通过 qume-info 查看 disk 文件的属性：

![http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20160501-1462109267370087342.png](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HdBCibgTzShNK4wxWvEGPKIw/0?wx_lazy=1)

这里有两个容易搞混淆的术语，在此特别说明一下：

1. image，指的是 Glance 上保存的镜像，作为 instance 运行的模板。 计算节点将下载的 image 存放在 /opt/stack/data/nova/instances/_base 目录下。
2. 镜像文件，指的是 instance 启动盘所对应的文件
3. 二者的关系是：image 是镜像文件 的 backing file。image 不会变，而镜像文件会发生变化。比如安装新的软件后，镜像文件会变大。

#### 创建 instance 的 XML 定义文件

创建的 XML 文件会保存到该 instance 目录 /opt/stack/data/nova/instances/f1e22596-6844-4d7a-84a3-e41e6d7618ef，命名为 libvirt.xml

#### 创建虚拟网络并启动 instance

![http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20160501-1462109268755029101.png](http://mmsns.qpic.cn/mmsns/Hia4HVYXRicqFQtJ8pZe5tCJiaY2ibNv6Z9HtmIzRM8MjZuK6j3RvQhlqA/0?wx_lazy=1)

本环境用的是 linux-bridge 实现的虚拟网络。

CLI 查看实例：

```
virsh list
```

