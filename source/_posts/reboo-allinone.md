---
title: 重新启动服务器后，devstack 的启动
date: 2018-01-11 10:35:57
tags: [OpenStack,devstack]
categories: SRE
---

通过 devstack 方式安装的 all-in-one，如果在服务器重启之后，很多 OpenStack 服务是起不来的，这个时候需要执行一些命令。



![OpenStack 服务启动失败](http://ww1.sinaimg.cn/large/c552abe7ly1fncp0uceu7j20hv05pgmy.jpg)

```shell
sudo su - stack

sudo losetup -f /opt/stack/data/stack-volumes-default-backing-file
sudo losetup -f /opt/stack/data/stack-volumes-lvmdriver-l-backing-file
script /dev/null
screen -c /opt/stack/devstack/stack-screenrc

ctrl + A，然后 D 退出
```

