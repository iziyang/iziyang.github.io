---
title: 手把手教你安装 OpenStack-Ocata 安装指南
date: 2017-03-14 09:50:34
tags:
categories: SRE
---
本文参考：[https://docs.openstack.org/ocata/install-guide-rdo/index.html](https://docs.openstack.org/ocata/install-guide-rdo/index.html) 官方文档来手把手教你安装 Ocata，安装文档中有漏洞的地方，本文都会提及。

本文目录：

一、	安装环境

二、	认证服务

三、	镜像服务

四、	计算服务

五、	网络服务

六、	仪表盘




# 一、	安装环境 #

## 1、示例架构 ##

根据官方文档，本文架构采用一个控制节点和一个计算节点。


> （The example architecture requires at least two nodes (hosts) to launch a basic virtual machine or instance. ）

控制节点运行认证服务、镜像服务、计算服务的管理部分、网络服务的管理部分、各种网络代理以及Dashboard，还包括一些基础服务如数据库、消息队列以及NTP。

计算节点上运行计算服务中管理实例的管理程序部分。默认情况下，计算服务使用 KVM。还运行网络代理服务，用来连接实例和虚拟网络以及通过安全组给实例提供防火墙服务。

## 2、网络 ##


### 1) 公有网络

公有网络选项以尽可能简单的方式通过layer-2（网桥/交换机）服务以及VLAN网络的分割来部署OpenStack网络服务。实际上，它将虚拟网络桥接到物理网络，并依靠物理网络基础设施提供layer-3服务(路由)。另外，DHCP服务为实例提供IP地址。

### 2) 私有网络

私有网络选项扩展了公有网络选项，增加了layer-3（路由）服务，使用VXLAN类似的方式。本质上，它使用NAT路由虚拟网络到物理网络。另外，这个选项也提供高级服务的基础，比如LBaas和FWaaS。

这里我们选择私有网络。

## 3、安全 ##

下面是各个需要密码的服务以及解释。

![alt text](/images/hand1.png)

## 4、主机网络配置 ##

### 1)	控制节点

**配置网络接口**

    Controller；IP：192.168.0.112/24 GATEWAY：192.168.0.1

网络配置完之后需要将防火墙和selinux关闭。

	关闭防火墙：systemctl stop firewalld.service
	禁止防火墙开机启动；systemctl disable firewalld.service
	关闭selinux：vim /etc/selinux/config selinux=disabled 
	setenforce 0 使配置立即生效

**配置地址解析**

编辑`/etc/hosts`

	# controller
	192.168.0.112       controller
	# compute1
	192.168.0.113       compute1

### 2) 计算节点

**配置网络接口**

    Compute：IP：192.168.0.113/24 GATEWAY：192.168.0.1

同样需要配置地址解析，参考控制节点。

配置完成之后需要进行验证：

1、与外网连通性：

分别在控制节点和计算节点上：

    # ping -c 4 www.baidu.com

控制节点和计算节点连通性：

	控制节点上输入# ping -c 4 compute1
	计算节点上输入# ping -c 4 controller

## 5、NTP ##

### 1) 控制节点

**安装相关包**

    # yum install chrony

编辑 `/etc/chrony.conf`

    server NTP_SERVER iburst

可以根据需要将NTP_SERVER替换为合适的NTP服务器，建议不用改。
然后添加：

    allow 192.168.0.0/24

即允许计算节点同步。（计算节点IP网段属于192.168.0.0）

	# systemctl enable chronyd.service
	# systemctl start chronyd.service

启动服务。

### 2) 计算节点

编辑`/etc/chrony.conf`，可以将其他的几个地址注释掉。

    server controller iburst

同步控制节点：

	# systemctl enable chronyd.service
	# systemctl start chronyd.service

启动服务。

> 如果发现同步的不是控制节点，那么重启一下NTP服务即可。

验证操作：

在控制节点上同步时间：

	# chronyc sources

带星号的是NTP当前同步的地址。

计算节点上同步。

	# chronyc sources
	210 Number of sources = 1
	MS Name/IP address         Stratum Poll Reach LastRx Last sample
	===========================================================================
  	^* controller            3    9   377   421    +15us[  -87us] +/-   15ms

## 6、安装OpenStack包 ##

以下操作在所有节点上进行。

启用OpenStack库：

	# yum install centos-release-openstack-ocata

完成安装：

1.在所有节点上升级包

	# yum upgrade

2.安装OpenStack 客户端

	# yum install python-openstackclient

3.CentOS默认启用了SELinux，安装openstack-selinux来自动管理OpenStack服务的安全策略。

	# yum install openstack-selinux

## 7、安装数据库 ##

数据库一般运行在控制节点上。

安装并配置组件：

１、安装相关包：

	# yum install mariadb mariadb-server python2-PyMySQL

２、创建并编辑/etc/my.cnf.d/openstack.cnf 文件，并完成以下操作。

在配置文件中添加以下配置：

	[mysqld]
	bind-address = 192.168.0.112
	default-storage-engine = innodb
	innodb_file_per_table = on
	max_connections = 4096
	collation-server = utf8_general_ci
	character-set-server = utf8
	其中bind-address为控制节点IP地址。

完成安装：

１、启动数据库并设置开机启动：

	# systemctl enable mariadb.service
	# systemctl start mariadb.service

２、运行mysql_secure_installation脚本来保证数据库安全，为root账户设置一个合适的密码：

	# mysql_secure_installation

## 8、消息队列 ##

OpenStack使用消息队列来协调服务之间的状态和操作，消息队列服务一般运行在控制节点上。OpenStack支持RabbitMQ，Qpid以及ZeroMQ等消息队列服务。本指南使用RabbitMQ消息队列服务。

**安装并配置组件：**

１、安装相关包

	# yum install rabbitmq-server

２、启动消息队列并设置开机启动

	# systemctl enable rabbitmq-server.service
	# systemctl start rabbitmq-server.service

３、添加openstack用户

	# rabbitmqctl add_user openstack RABBIT_PASS
	Creating user "openstack" ...

使用合适的密码替换掉`RABBIT_PASS `

４、允许openstack用户的配置、写、读权限：

    # rabbitmqctl set_permissions openstack ".*" ".*" ".*"
    Setting permissions for user "openstack" in vhost "/" ...

## 9、缓存令牌 ##

认证服务的认证机制使用Memcached来缓存令牌，一般运行在控制节点上。

**安装并配置组件**

1、安装相关包

    # yum install memcached python-memcached

2、编辑 /etc/sysconfig/memcached文件并配置IP地址，将127.0.0.1改为控制节点IP

3、	完成安装

启动 Memcached服务并设置开机启动

	# systemctl enable memcached.service
	# systemctl start memcached.service

# 二、	认证服务 #

## 1、在控制节点上配置 ##

**前提条件**

创建数据库：

以root身份登录数据库

    $ mysql -u root -p

创建keystone数据库

    MariaDB [(none)]> CREATE DATABASE keystone;

给数据库赋予适当的权限

	MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
	IDENTIFIED BY 'KEYSTONE_DBPASS';
	MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
	IDENTIFIED BY 'KEYSTONE_DBPASS';

用合适的密码替换`KEYSTONE_DBPASS`

## 2、安装并配置组件 ##

运行命令安装相关包

	# yum install openstack-keystone httpd mod_wsgi

编辑文件`/etc/keystone/keystone.conf`

在`[database]`选项配置数据库连接

	[database]
	# ...
	connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

替换掉`KEYSTONE_DBPASS`

在`[token]`选项中，配置Fernet令牌提供者

    [token]
    # ...
    provider = fernet

1、同步认证服务数据库

	# su -s /bin/sh -c "keystone-manage db_sync" keystone

2、初始化Fernet key仓库

	# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
	# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

3、引导认证服务

	# keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
	  --bootstrap-admin-url http://controller:35357/v3/ \
	  --bootstrap-internal-url http://controller:5000/v3/ \
	  --bootstrap-public-url http://controller:5000/v3/ \
	  --bootstrap-region-id RegionOne

替换掉 `ADMIN_PASS `

## 3、配置Apache服务器 ##

1、编辑 `/etc/httpd/conf/httpd.conf`并配置`ServerName`选项，使之参考控制节点

2、给`/usr/share/keystone/wsgi-keystone.conf`文件创建一个链接

	# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

**完成安装**

1、启动Apache服务器并设置开机启动

	# systemctl enable httpd.service
	# systemctl start httpd.service

2、配置管理账户

	$ export OS_USERNAME=admin
	$ export OS_PASSWORD=ADMIN_PASS
	$ export OS_PROJECT_NAME=admin
	$ export OS_USER_DOMAIN_NAME=Default
	$ export OS_PROJECT_DOMAIN_NAME=Default
	$ export OS_AUTH_URL=http://controller:35357/v3
	$ export OS_IDENTITY_API_VERSION=3


**创建一个域、项目、用户和角色**

1、本指南有一个service 项目，你添加的每一个服务都有唯一的用户。

	$ openstack project create --domain default \
	  --description "Service Project" service

2、普通的任务不应该使用具有特权的项目和用户。作为示例，本指南创建一个demo项目和用户。

创建demo项目：

	$ openstack project create --domain default \
	  --description "Demo Project" demo

创建demo用户：

	$ openstack user create --domain default \
	  --password-prompt demo

创建user角色：

	$ openstack role create user

将user角色添加到demo项目和用户中。

	$ openstack role add --project demo --user demo user

## 4、验证操作 ##

1、	出于安全性的原因，禁用掉暂时的认证令牌机制。

编辑`/etc/keystone/keystone-paste.ini`文件，并从`[pipeline:public_api], [pipeline:admin_api]`, 和 `[pipeline:api_v3]`选项中删除`admin_token_auth`

![alt text](/images/hand2.png)

2、	取消设置临时的`OS_AUTH_URL`和 `OS_PASSWORD`环境变量

    $ unset OS_AUTH_URL OS_PASSWORD

3、	使用admin用户，请求一个认证令牌

	$ openstack --os-auth-url http://controller:35357/v3 \
	--os-project-domain-name default --os-user-domain-name default \
	--os-project-name admin --os-username admin token issue

这里遇到错误

![alt text](/images/hand3.png)

由于是Http错误，所以返回Apache HTTP 服务配置的地方
重启Apache 服务，并重新设置管理账户：

	# systemctl restart httpd.service
	$ export OS_USERNAME=admin
	$ export OS_PASSWORD=ADMIN_PASS
	$ export OS_PROJECT_NAME=admin
	$ export OS_USER_DOMAIN_NAME=Default
	$ export OS_PROJECT_DOMAIN_NAME=Default
	$ export OS_AUTH_URL=http://controller:35357/v3
	$ export OS_IDENTITY_API_VERSION=3

![alt text](/images/hand4.png)

4、	使用demo用户，请求认证令牌：

	$ openstack --os-auth-url http://controller:5000/v3 \
	  --os-project-domain-name default --os-user-domain-name default \
	  --os-project-name demo --os-username demo token issue
	
	Password:

密码为创建demo用户时的密码。

## 5、创建OpenStack客户端环境脚本 ##

在前面章节中，我们使用环境变量和命令的组合来配置认证服务，为了更加高效和方便，我们创建一个脚本方便以后的操作。这些脚本包括一些公共的操作，但是也支持自定义的操作。

### 1、创建脚本
创建并编辑 `admin-openrc`文件，并添加以下内容：

	export OS_PROJECT_DOMAIN_NAME=Default
	export OS_USER_DOMAIN_NAME=Default
	export OS_PROJECT_NAME=admin
	export OS_USERNAME=admin
	export OS_PASSWORD=root
	export OS_AUTH_URL=http://controller:35357/v3
	export OS_IDENTITY_API_VERSION=3
	export OS_IMAGE_API_VERSION=2

替换掉`OS_PASSWORD`的密码。

创建并编辑`demo-openrc`文件，并添加以下内容：

	export OS_PROJECT_DOMAIN_NAME=Default
	export OS_USER_DOMAIN_NAME=Default
	export OS_PROJECT_NAME=demo
	export OS_USERNAME=demo
	export OS_PASSWORD=DEMO_PASS
	export OS_AUTH_URL=http://controller:5000/v3
	export OS_IDENTITY_API_VERSION=3
	export OS_IMAGE_API_VERSION=2

### 2、使用脚本

加载脚本文件更新环境变量：

    $ . admin-openrc

请求一个认证令牌：

    $ openstack token issue

下文待续。
