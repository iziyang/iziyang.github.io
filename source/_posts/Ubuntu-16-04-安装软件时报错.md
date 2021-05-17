---
title: Ubuntu 16.04 安装软件时报错
date: 2017-03-28 20:07:31
tags: Linux
categories: 编程
---
# 方法一 #
今天本来想用Tkinter做图形界面，可是却提示说没有这个库，于是乎去安装Tkinter，可是又遇到报错：

    beizi@beizi-virtual-machine:~$ sudo apt-get install python-tk
    [sudo] beizi 的密码： 
    E: 无法获得锁 /var/lib/dpkg/lock - open (11: 资源暂时不可用)
    E: 无法锁定管理目录(/var/lib/dpkg/)，是否有其他进程正占用它？
    

于是在网上查找解决办法：

    ps aux | grep "apt-get"

列出包括apt-get的进程，然后kill掉该进程，重新安装即可。

    sudo kill 3213

# 方法二 #

删除掉该目录：

    sudo rm var/lib/dpkg/lock

根据测试，方法一更为有效。

