---
title: Newton 配置 Spice 协议
date: 2017-03-14 16:23:44
tags:
categories: SRE
---
# 1、控制节点 #

## 安装 ##

	# yum install spice-server spice-protocol openstack-nova-spicehtml5proxy spice-html5



这里提示：

![alt text](/images/spice1.png)

直接下载`spice-html5`的RPM包并安装：

	# wget http://dl.fedoraproject.org/pub/epel/7/x86_64/s/spice-html5-0.1.7-1.el7.noarch.rpm
	# rpm -i spice-html5-0.1.7-1.el7.noarch.rpm

## 配置 ##

修改配置文件

	# vim /etc/nova/nova.conf
	[default]
	vnc_enabled=false
	[spice]
	html5proxy_host=0.0.0.0
	html5proxy_port=6082
	keymap=en-us

停止novncproxy并取消自启动

	# systemctl stop openstack-nova-novncproxy.service
	# systemctl disable openstack-nova-novncproxy.service

启用spicehtml5proxy开机自启动并启动它

	# systemctl enable openstack-nova-spicehtml5proxy.service
	# systemctl start openstack-nova-spicehtml5proxy.service

# 2、计算节点 #

## 安装 ##

	# yum install spice-server spice-protocol spice-html5

这里同样提示找不到软件包spice-html5，参照控制节点安装方法。

## 配置 ##

修改配置文件

    # vim /etc/nova/nova.conf
    [default]
    vnc_enabled=false
    [vnc]
    enabled = False
    [spice]
    html5proxy_base_url=
    server_listen=127.0.0.1
    server_proxyclient_address=127.0.0.1
    enabled=true
    keymap=en-us

重启nova-compute

    # systemctl restart openstack-nova-compute.service

但是通过dashboard并没有看到控制台。

![alt text](/images/spice2.png)

这里发现实例是关机状态，重启实例后，在控制台发现：127.0.0.1拒绝了连接请求，因此参考vnc选项的配置，将IP修改如下：

	[spice]

	html5proxy_base_url=http://controller:6082/spice_auto.html
	server_listen=$my_ip
	server_proxyclient_address=$my_ip
	enabled=true
	keymap=en-us

将url修改为controller，其他IP修改为$my_ip
再次查看控制台：

![alt text](/images/spice4.png)

安装成功！



