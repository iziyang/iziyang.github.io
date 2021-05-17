---
title: LVM 配置
date: 2017-10-09 09:58:12
tags: [Linux,LVM,分区]
categories: 编程
---

# LVM 结构图

<!-- more -->

![](http://ww1.sinaimg.cn/large/c552abe7ly1fkbsqqnikdj20gy06vaa6.jpg)

# 空白磁盘的 LVM 配置

1、parted 查看磁盘的未分区空间

![](http://ww1.sinaimg.cn/large/c552abe7ly1fkbtiik0njj215y0d1q3u.jpg)

2、fdisk 对空白磁盘进行分区（也可使用 parted 命令分区）

3、fdisk 将 systemID 修改为LVM标记（8e）

4、partprobe 更新系统分区表

![](http://ww1.sinaimg.cn/large/c552abe7ly1fkbtw880dvj20ml0k7q4m.jpg)

5、pvcreat 将 Linux 分区处理成物理卷 PV

6、vgcreate 将 PV 处理成 VG，或者 vgextend，将某个物理卷添加进 VG

7、lvcreate 将卷组分成若干个逻辑卷 LV

8、mkfs.xfs 将 LV 格式化，挂载格式化后的 LV 到文件系统

# 管理文件系统空间

## 增大空间

1. umount 先卸载逻辑卷
2. 通过 vgextend，lvextend 增大 LV 空间
3. 再使用 resize2fs 将逻辑卷容量增加
4. mount 重新挂载

## 减小空间

1. umount 卸载逻辑卷
2. 使用 resize2fs 将逻辑卷容量减小
3. vgreduce，lvreduce 减小 LV
4. mount 重新挂载

# LVM 管理命令

## 物理卷（PV）管理

```shell
pvcreate 创建物理卷 # pvcreate /dev/sda6
pvremove 删除物理卷
pvscan 查看物理卷信息
pvdisplay 查看各个物理卷的详细参数
```

## 卷组（VG）管理

```shell
vgcreate 创建卷组
vgscan 查看卷组信息
vgdisplay 查看卷组详细参数
vgreduce 缩小卷组，把某个物理卷删除
vgextend 扩展卷组，把某个物理卷添加到卷组
vgremove 删除卷组
```

## 逻辑卷（LV）管理

```shell
lvcreate 创建逻辑卷
lvscan 查看逻辑卷信息
lvdisplay 查看逻辑卷具体参数
lvextend 增大逻辑卷大小
lvreduce 减小逻辑卷大小
lvremove 删除逻辑卷
lvs 查看逻辑卷属性信息
```

## 快照管理

### 创建快照

创建快照需要注意：

1. VG 中需要预留存放快照本身的空间，不能全部被占满。
2. 快照所在的 VG 必须与被备份的 LV 相同，也就是说，快照存放的位置必须与被照卷存放在同一个 VG 上。否则快照会失败。

为根分区创建快照：

```shell
[root@controller ~]# lvcreate -L 10G -s -n root_orign_backup /dev/centos_controller/root 
  Using default stripesize 64.00 KiB.
  Logical volume "root_orign_backup" created.
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1fkcxyo92yoj20ur05h0sy.jpg)

### 恢复快照

要恢复快照，我们首先需要卸载文件系统。注意，要卸载被恢复的文件系统。

只想检查挂载点是否卸载成功，可以使用下面的命令。

```shell
df -h
```

卸载之后恢复快照：

```shell
# lvconvert --merge /dev/centos_controller/root_orign_backup
```

在合并完成后，快照卷将被自动移除。

### 自动扩展

要自动扩展快照，我们可以通过修改配置文件来进行。对于手动扩展，我们可以使用lvextend。

```
# vim /etc/lvm/lvm.conf
```

搜索单词 autoextend。

![](http://ww1.sinaimg.cn/large/c552abe7ly1fkd0gohi50j20k108haab.jpg)

修改此处的100为75，这样自动扩展的起始点就是75，而自动扩展百分比为20，它将自动扩容百分之20。

# 一些磁盘管理命令

lsblk：查看分区情况

```shell
[root@controller ~]# lsblk 
NAME                       MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                          8:0    0 153.4G  0 disk 
├─sda1                       8:1    0     1G  0 part /boot
├─sda2                       8:2    0 115.6G  0 part 
│ ├─centos_controller-root 253:0    0   100G  0 lvm  /
│ ├─centos_controller-swap 253:1    0   5.6G  0 lvm  [SWAP]
│ └─centos_controller-home 253:2    0    10G  0 lvm  /home
└─sda3                       8:3    0  36.8G  0 part                
```

df -h：查看可用空间

```shell
[root@controller ~]# df -h
Filesystem                          Size  Used Avail Use% Mounted on
/dev/mapper/centos_controller-root  100G 1022M   99G   1% /
devtmpfs                            2.7G     0  2.7G   0% /dev
tmpfs                               2.7G     0  2.7G   0% /dev/shm
tmpfs                               2.7G  8.6M  2.7G   1% /run
tmpfs                               2.7G     0  2.7G   0% /sys/fs/cgroup
/dev/sda1                          1014M  143M  872M  15% /boot
/dev/mapper/centos_controller-home   10G   33M   10G   1% /home
tmpfs                               551M     0  551M   0% /run/user/0
```

fdisk：分区命令

```shell
[root@controller ~]# fdisk -l

Disk /dev/sda: 164.7 GB, 164696555520 bytes, 321672960 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0000bfc8

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   244592639   121246720   8e  Linux LVM
/dev/sda3       244592640   321672959    38540160   8e  Linux LVM

Disk /dev/mapper/centos_controller-root: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos_controller-swap: 6039 MB, 6039797760 bytes, 11796480 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos_controller-home: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

parted：与fdisk类似

```shell
[root@controller ~]# parted
GNU Parted 3.1
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) help                                                             
  align-check TYPE N                        check partition N for TYPE(min|opt) alignment
  help [COMMAND]                           print general help, or help on COMMAND
  mklabel,mktable LABEL-TYPE               create a new disklabel (partition table)
  mkpart PART-TYPE [FS-TYPE] START END     make a partition
  name NUMBER NAME                         name partition NUMBER as NAME
  print [devices|free|list,all|NUMBER]     display the partition table, available devices, free space, all found partitions, or a particular partition
  quit                                     exit program
  rescue START END                         rescue a lost partition near START and END
  rm NUMBER                                delete partition NUMBER
  select DEVICE                            choose the device to edit
  disk_set FLAG STATE                      change the FLAG on selected device
  disk_toggle [FLAG]                       toggle the state of FLAG on selected device
  set NUMBER FLAG STATE                    change the FLAG on partition NUMBER
  toggle [NUMBER [FLAG]]                   toggle the state of FLAG on partition NUMBER
  unit UNIT                                set the default unit to UNIT
  version                                  display the version number and copyright information of GNU Parted

```

