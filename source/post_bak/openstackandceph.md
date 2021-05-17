---
title: OpenStack 集成 Ceph
date: 2017-10-27 09:47:50
tags: [Ceph,OpenStack]
categories: SRE
---

# 创建存储池

> 我这里选择 glance 使用控制节点的存储，VM 保存在计算节点上，备份没有配置，所以 Ceph 只作为卷使用。

在 Ceph 节点上面，创建存储池：

<!-- more -->

```shell
[root@node1 ~]# ceph osd pool create volumes 128
pool 'volumes' created
```

> 根据官方文档，这里初始化存储池失败。
>
> ```shell
> [root@node1 ~]# rbd pool init volumes
> error: unknown option 'pool init volumes'
> ```

# 配置 OpenStack 为 Ceph 客户端

运行 `glance-api`, `cinder-volume`, `nova-compute` , `cinder-backup` 进程的节点都需要 Ceph 客户端。每一个都要有 `ceph.conf` 文件。

在计算节点和控制节点上创建 Ceph 目录：

```shell
[root@controller ~]# cd /etc
[root@controller etc]# mkdir ceph
```

将配置文件传到控制节点和计算节点：

```shell
[root@node1 ~]# ssh root@compute01 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
```

我部署的 OpenStack 为一个控制节点和一个计算节点。 `glance-api` 运行在控制节点上，`cinder-volume`, `nova-compute` 运行在计算节点上，因此，控制节点和计算节点都需要配置为 Ceph 客户端。

## 安装 Ceph 客户端安装包

在 `glance-api` 节点（控制节点）上，需要安装 librbd 的 Python 绑定：

```shell
[root@controller ~]# sudo yum install python-rbd
```

在 `cinder-volume`, `nova-compute` 节点（计算节点）上，安装 Python 绑定以及 Ceph 命令行工具：

```shell
[root@compute01 ~]# sudo yum install ceph-common
[root@controller ~]# sudo yum install python-rbd
```

## 配置 Ceph 客户端授权

从 Ceph monitor 节点，为 Cinder 创建一个新用户。

```
[root@node1 ~]# ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes'
[client.cinder]
	key = AQDXCvRZ+OpCFBAANnmyibi/4vtZtyu5HJcQvw==
```

为 client.cinder 添加密钥环并改变所属者。

```shell
[root@node1 ~]# ceph auth get-or-create client.cinder | ssh root@compute01 sudo tee /etc/ceph/ceph.client.cinder.keyring
root@compute01's password: 
[root@node1 ~]# ssh root@compute01 chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
root@compute01's password:
```

运行 nova-compute 的节点还需要将 client.cinder 用户的密钥存储在 libvirt 中。在从 Cinder 中添加块设备时，libvirt 进程需要它访问群集。

在运行 nova-compute 的节点上创建密钥的临时副本：

```
[root@node1 ~]# ceph auth get-key client.cinder | ssh root@compute01 tee client.cinder.key
```

在计算节点上，为密钥生成 UUID, 并保存 UUID，以便待会配置 nova-compute。

```
[root@compute01 ~]# uuidgen > uuid-secret.txt
```

然后在计算节点上，向 libvirt 中添加密钥，并删除临时副本：

```
[root@compute01 ~]# cat > secret.xml <<EOF
> <secret ephemeral='no' private='no'>
>   <uuid>`cat uuid-secret.txt`</uuid>
>   <usage type='ceph'>
>     <name>client.cinder secret</name>
>   </usage>
> </secret>
> EOF

[root@compute01 ~]# virsh secret-define --file secret.xml
[root@compute01 ~]# virsh secret-set-value --secret $(cat uuid-secret.txt) --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml
```

# 配置 Cinder

 `cinder-volume` 节点需要 Ceph 块设备，volume 存储池，用户以及密钥的 UUID，才能和 Ceph 块设备连接。要配置 Cinder，请执行以下步骤：

1. 打开 Cinder 配置文件。

   ```
   [root@compute01 ~]# vim /etc/cinder/cinder.conf
   ```

2. 在 `[DEFAULT]` 选项中，开启 Ceph 作为 Cinder 后端存储。

   ```
   enabled_backends = ceph
   ```

3. 确认 Glance API 版本是 2。如果你在`enabled_backends` 中配置了多后端， `glance_api_version = 2` 必须在 `[DEFAULT]` 选项中设置而不是在 `[ceph]` 选项中。

   ```
   glance_api_version = 2
   ```

4. 在`cinder.conf` 中创建一个 `[ceph]` 选项，添加以下部分到 `[ceph]` 选项中。

5. 设置 `volume_driver` 使用 Ceph 后端存储。

   ```
   volume_driver = cinder.volume.drivers.rbd.RBDDriver
   ```

6. 设置集群名称以及 Ceph 配置文件的位置。

   ```
   rbd_cluster_name = e8b7ae85-8d03-4160-becc-10df8f385c59
   rbd_ceph_conf = /etc/ceph/ceph.conf
   ```

7. 默认情况下，OSP 在 `rbd` 池中存储 Ceph 卷。

   ```
   rbd_pool = volumes
   ```

8. 指定 `rbd_user` ，把它设置为 `cinder` 用户。然后指定 `rbd_secret_uuid` ，UUID 存储在 `uuid-secret.txt` 文件中。

   ```
   rbd_user = cinder
   rbd_secret_uuid = 0fb0bd0a-7926-4c4a-b015-b1cfa3f86456
   ```

9. 指定以下设置。

   ```
   rbd_flatten_volume_from_snapshot = false
   rbd_max_clone_depth = 5
   rbd_store_chunk_size = 4
   rados_connect_timeout = -1
   ```

最终的设置应该像下面一样：

```
[DEFAULT]
enabled_backends = ceph
glance_api_version = 2
···
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_cluster_name = e8b7ae85-8d03-4160-becc-10df8f385c59
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_pool = volumes
rbd_user = cinder
rbd_secret_uuid = 0fb0bd0a-7926-4c4a-b015-b1cfa3f86456
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
```

# 重启 OpenStack 服务

```
[root@compute01 ~]# systemctl restart openstack-cinder-volume
[root@compute01 ~]# systemctl restart openstack-nova-compute
```

# 参考

[https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/2/html-single/ceph_block_device_to_openstack_guide/#restarting_openstack_services](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/2/html-single/ceph_block_device_to_openstack_guide/#restarting_openstack_services)

官方文档基本没用。