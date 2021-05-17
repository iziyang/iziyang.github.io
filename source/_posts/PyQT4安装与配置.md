---
title: PyQT 及 Eric 的安装与配置
date: 2017-05-18 14:30:12
tags: [Python,PyQT]
categories: 编程
---

> PyQT4的安装依赖于SIP，因此要先安装配置SIP。另外，GUI界面的开发要用到eric，因此也需要配置安装。

> 系统环境为Win10

# 安装SIP

1. **下载SIP源码**。[http://www.riverbankcomputing.com/software/sip/download](http://www.riverbankcomputing.com/software/sip/download)
2. **配置SIP。**在SIP文件夹下(最好放在Python2.7\Lib\site-packages\)。输入：

   python configure.py



由于Windows没有make命令，因此在VS的命令行（使用的VS2015 x86 x64兼容工具提示符）中输入（幸好本机安装了VS2015）：

	nmake
	nmake install

![alt text](/images/SIP安装.png)

![alt text](/images/SIPnmakeinstall.png)

# 安装PyQT4

1. **下载PyQT4。**[http://www.riverbankcomputing.com/software/pyqt/download](http://www.riverbankcomputing.com/software/pyqt/download)
2. **配置PyQT4。**同样在相应文件夹下输入：

   python configure-ng.py

报错：

    Error: Make sure you have a working Qt qmake on your PATH.

然后多方查询，发现应该是需要先安装QT，于是下载QT安装（注册、下载，各种麻烦）

最终由于过程太过繁琐，放弃安装PyQT4。

# PyQT5安装

首先安装了Python3，然后使用pip安装：

	py -3 -m pip install pyqt5

这里说明一下Python2和Python3的使用方法：

	py -2
	py -3
	安装软件使用
	py -* -m pip

# Eric6安装与配置

在安装好Python3和PyQT5之后，还需要安装：

	py -3 -m pip install qscintilla

然后执行：
	py -3 install.py

安装完成之后需要进行配置：

点击编辑器-->API，语言选择Python3，然后从已有API导入，编译API，OK。

但是这里依然有问题：

我创建窗体时提示：

![alt text](/images/qtdesigner.png)

最终我选择卸载Python3.6，卸载PyQT5.8，重新安装Python3.5，PyQT5.6，而且是直接下载安装包安装。再重新安装Eric6。

但是安装完Eric6又出现问题，直接无法启动，说调试器后端无法启动，百度得知，需要通过管理员权限的命令行提示符来安装。

进入命令行提示符却怎么也无法切换到D盘，最终：

    cd /d d:

进入了D盘，安装完了Eric6。

参考链接：

[http://pyqt.sourceforge.net/Docs/sip4/installation.html](http://pyqt.sourceforge.net/Docs/sip4/installation.html)

[http://pyqt.sourceforge.net/Docs/PyQt4/installation.html](http://pyqt.sourceforge.net/Docs/PyQt4/installation.html)

[http://pyqt.sourceforge.net/Docs/PyQt5/installation.html](http://pyqt.sourceforge.net/Docs/PyQt5/installation.html)

[http://flyfowl.com/2016/08/10/windows%E4%B8%8B%E5%AE%89%E8%A3%85eric6-python-ide.html](http://flyfowl.com/2016/08/10/windows%E4%B8%8B%E5%AE%89%E8%A3%85eric6-python-ide.html "无法启动QTdesigner的解决办法")

[https://sourceforge.net/projects/pyqt/files/PyQt5/](https://sourceforge.net/projects/pyqt/files/PyQt5/ "PyQT下载链接")