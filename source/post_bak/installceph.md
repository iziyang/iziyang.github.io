---
title: 安装 Ceph 集群
date: 2017-10-10 17:09:44
tags: [Ceph,OpenStack]
categories: SRE
---

# 预检

ceph 集群采用如下的架构：

<!-- more -->

![](http://ww1.sinaimg.cn/large/c552abe7ly1fkoj8qjfmsj20eg08kmx5.jpg)

## 安装 Ceph 部署工具（管理节点）

> 添加向管理节点上添加 Ceph 仓库，然后安装 ceph-deploy 工具。

1. 添加并启用 EPEL  仓库：

   ```shell
   [root@admin-node ~]# yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
   ```


2. 向 yum 配置文件中添加 Ceph 仓库，yum 配置文件在 `/etc/yum.repos.d/ceph.repo` 。

   ```shell
   [root@admin-node ~]# vim /etc/yum.repos.d/ceph.repo

   [ceph-noarch]
   name=Ceph noarch packages
   baseurl=https://download.ceph.com/rpm/el7/noarch
   enabled=1
   gpgcheck=1
   type=rpm-md
   gpgkey=https://download.ceph.com/keys/release.asc
   ```

3. 更新仓库并安装 ceph-deploy 工具：

   ```shell
   [root@admin-node ~]# yum update
   [root@admin-node ~]# yum install ceph-deploy
   ```

## Ceph 节点安装

### 安装 NTP

```shell
[root@ceph_node ~]# yum install ntp ntpdate ntp-doc
[root@ceph_node ~]# ntpdate pool.ntp.org
[root@ceph_node ~]# systemctl restart ntpdate
[root@ceph_node ~]# systemctl restart ntpd
[root@ceph_node ~]# systemctl enable ntpd
[root@ceph_node ~]# systemctl enable ntpdate
```

### 安装 SSH 服务器

```shell
[root@ceph_node ~]# sudo yum install openssh-server
[root@ceph_node ~]# systemctl status sshd # 确保 SSH 服务器在所有 Ceph 节点上运行
```

### 创建一个 Ceph 部署用户

1. 在每个 Ceph 节点上创建一个新用户（storage）：

   ```shell
   [root@ceph_node ~]# useradd -d /home/storage -m storage
   [root@ceph_node ~]# passwd storage
   ```

2. 确保用户具有 sudo 特权：

   ```shell
   [root@ceph_node ~]# echo "storage ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/storage
   [root@ceph_node ~]# sudo chmod 0440 /etc/sudoers.d/storage
   ```

### 开启无密码 SSH（管理节点）

1. 生成 SSH 密钥，但是不要用 sudo 或者 root 用户。口令保留为空。

   ```shell
   [storage@admin-node ~]$ ssh-keygen
   Generating public/private rsa key pair.
   Enter file in which to save the key (/home/storage/.ssh/id_rsa): 
   Created directory '/home/storage/.ssh'.
   Enter passphrase (empty for no passphrase): 
   Enter same passphrase again: 
   Your identification has been saved in /home/storage/.ssh/id_rsa.
   Your public key has been saved in /home/storage/.ssh/id_rsa.pub.
   ```

2. 复制密钥到各个 Ceph 节点。

   ```shell
   [storage@admin-node ~]$ ssh-copy-id storage@controller
   [storage@admin-node ~]$ ssh-copy-id storage@compute01
   [storage@admin-node ~]$ ssh-copy-id storage@compute02
   ```

> 这里官方文档建议可以修改 `~/.ssh/config` 文件，把用户名写进去，这样在 ceph-deploy 时就不用指定用户名，事实证明，这个方法无效，会提示拒绝访问。还是指定用户名比较好。

### 关闭防火墙和 SELINUX

```shell
[root@ceph_node ~]# systemctl stop firewalld
[root@ceph_node ~]# systemctl disable firewalld
[root@ceph_node ~]# setenforce 0
[root@ceph_node ~]# vim /etc/selinux/config
SELINUX=disabled
```

### 安装优先软件包

```shell
[root@ceph_node ~]# yum install yum-plugin-priorities
```

# 快速安装存储集群

先在管理节点上创建一个目录，这样，所有生成的文件都会保存在该目录下。

```shell
[storage@admin-node ~]$ mkdir my-cluster
[storage@admin-node ~]$ cd my-cluster
```

不要用 `sudo` 或者 root 用户执行 ceph-deploy 命令，因为它不会在远程主机上调用所需的 `sudo` 命令。

## 从头开始

如果在某些地方遇到麻烦，可以执行下面的命令从头开始。

```shell
[storage@admin-node ~]$ ceph-deploy --username storage purge {ceph-node} [{ceph-node}]
[storage@admin-node ~]$ ceph-deploy --username storage purgedata {ceph-node} [{ceph-node}]
[storage@admin-node ~]$ ceph-deploy forgetkeys
[storage@admin-node ~]$ rm ceph*
```

## 创建集群

1. 创建集群。先初始化监控节点。

   ```shell
   [storage@admin-node my-cluster]$ ceph-deploy --username storage new controller
   ```

   查看输出 `ls` 你可以看到一个 Ceph 配置文件 `ceph.conf` 一个监控密钥环 `ceph.mon.keyring` 还有一个日志文件。

   ```shell
   [storage@admin-node my-cluster]$ ls
   ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
   ```

2. 安装 Ceph 软件包。

   ```shell
   [storage@admin-node my-cluster]$ ceph-deploy --username storage install controller compute01 compute02
   ```

3. 配置初始 monitor 并收集密钥环。

   ```shell
   [storage@admin-node my-cluster]$ ceph-deploy mon create-initial
   ```

   官方文档说会有以下输出：

   - ceph.client.admin.keyring
   - ceph.bootstrap-mgr.keyring
   - ceph.bootstrap-osd.keyring
   - ceph.bootstrap-mds.keyring
   - ceph.bootstrap-rgw.keyring
   - ceph.bootstrap-rbd.keyring

   但是我的输出是下面的：

   ```shell
   [storage@admin-node my-cluster]$ ls
   ceph.bootstrap-mds.keyring  ceph.client.admin.keyring  
   ceph.bootstrap-osd.keyring  
   ceph.bootstrap-rgw.keyring  
   ```

4. 复制配置文件和管理密钥到管理节点和 Ceph 节点，这样你就可以使用 ceph CLI ，而不用每次都需要指定 monitor 的地址和 ceph.client.admin.keyring 。

   ```shell
   [storage@admin-node my-cluster]$ ceph-deploy --username storage admin controller compute01 compute02
   ```

5. 添加三个 OSD。经过试验新版本的 Ceph OSD 进程只能装在空硬盘上，安装在目录上和分区上会失败。（据说之前的版本不需要，我装的 J 版本，10.2.10，但我试验了 H 版本，最终失败了，所以就选择在虚拟机里部署。）安装在空分区上经过如下步骤：

   ```shell
   [storage@admin-node my-cluster]$ ceph-deploy --username storage osd prepare controller:/dev/sda3 compute01:/dev/sda3 compute02:/dev/sda3
   [storage@admin-node my-cluster]$ ceph-deploy --username storage osd activate controller:/dev/sda3 compute01:/dev/sda3 compute02:/dev/sda3
   ```
   报错：

   ```shell
   [controller][WARNIN] ceph_disk.main.Error: Error: ['ceph-osd', '--cluster', 'ceph', '--mkfs', '--mkkey', '-i', u'2', '--monmap', '/var/lib/ceph/tmp/mnt.uidkdf/activate.monmap', '--osd-data', '/var/lib/ceph/tmp/mnt.uidkdf', '--osd-journal', '/var/lib/ceph/tmp/mnt.uidkdf/journal', '--osd-uuid', u'ade546d3-e1fa-4dd4-b5a5-f9fb8f145841', '--keyring', '/var/lib/ceph/tmp/mnt.uidkdf/keyring', '--setuser', 'ceph', '--setgroup', 'ceph'] failed : 2017-10-23 11:02:08.694483 7f69e5898800 -1 filestore(/var/lib/ceph/tmp/mnt.uidkdf) mkjournal error creating journal on /var/lib/ceph/tmp/mnt.uidkdf/journal: (13) Permission denied
   [controller][WARNIN] 2017-10-23 11:02:08.694495 7f69e5898800 -1 OSD::mkfs: ObjectStore::mkfs failed with error -13
   [controller][WARNIN] 2017-10-23 11:02:08.694536 7f69e5898800 -1  ** ERROR: error creating empty object store in /var/lib/ceph/tmp/mnt.uidkdf: (13) Permission denied
   [controller][WARNIN] 
   [controller][ERROR ] RuntimeError: command returned non-zero exit status: 1
   [ceph_deploy][ERROR ] RuntimeError: Failed to execute command: /usr/sbin/ceph-disk -v activate --mark-init systemd --mount /dev/sda3
   ```

6. 由于我的物理机只有一块硬盘，最终不得不选择在虚拟机中部署集群。在虚拟机中添加了一块硬盘，部署 OSD：

   > 注意，以下是我在虚拟机中重新按照之前的步骤部署的。主机名分别设置为 node1，node2，node3

   ```shell
   ceph-deploy osd create node1:vdb node2:vdb node3:vdb
   ```

   节点分区情况如下所示：

   ```shell
   [root@node1 ~]# lsblk
   NAME                        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
   sr0                          11:0    1 1024M  0 rom  
   vda                         252:0    0    9G  0 disk 
   ├─vda1                      252:1    0    1G  0 part /boot
   └─vda2                      252:2    0    8G  0 part 
     ├─centos_admin--node-root 253:0    0  7.1G  0 lvm  /
     └─centos_admin--node-swap 253:1    0  924M  0 lvm  [SWAP]
   vdb                         252:16   0  300G  0 disk 
   ├─vdb1                      252:17   0  295G  0 part /var/lib/ceph/osd/ceph-0
   └─vdb2                      252:18   0    5G  0 part
   ```

7. 检查集群的健康状态。

   ```shell
   [root@node1 ~]# ceph health
   HEALTH_OK
   [root@node1 ~]# ceph -s
       cluster e8b7ae85-8d03-4160-becc-10df8f385c59
        health HEALTH_OK
        monmap e1: 1 mons at {node1=192.168.0.211:6789/0}
               election epoch 4, quorum 0 node1
        osdmap e23: 3 osds: 3 up, 3 in
               flags sortbitwise,require_jewel_osds
         pgmap v48: 64 pgs, 1 pools, 0 bytes data, 0 objects
               101 MB used, 884 GB / 884 GB avail
                     64 active+clean
   ```

   ​