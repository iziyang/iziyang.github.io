---
title: Samba 服务器搭建及配置修改
date: 2018-04-08 22:20:57
tags: Linux
categories: 编程
---

ubuntu server 16.04 下安装 samba：



```shell
sudo apt-get install samba
```

安装完成后，可以先修改配置文件：

```shell
vim /etc/samba/smb.conf
# 共享家目录
[homes]
   comment = Home Directories
   browseable = ye
   read only = no
   create mask = 0666 # 保证在windows端读写文件时，不会变为可执行文件
   directory mask = 0666
   valid users = %S # 必须是系统用户才行
# 添加Samba用户
sudo smbpasswd -a user
# 启用用户
sudo smbpasswd -e user
# 修改完成后重启
sudo service smbd restart
```

windows 连接时，只需要输入`\\IP 地址\user` ，输入凭据即可。