---
title: 一、PyQT5 介绍
date: 2017-05-19 14:21:48
tags: [Python,PyQT]
categories: 编程
---

> 注：本教程翻译自[http://zetcode.com/gui/pyqt5/](http://zetcode.com/gui/pyqt5/)

# 关于PyQT5

PyQT5是来自Digia的Python和QT5应用程序框架绑定的集合。它可用于Python2和Python3。本教程使用Python3。QT库是最强大的GUI库。PyQt5的官方网站是[http://www.riverbankcomputing.co.uk/news](http://www.riverbankcomputing.co.uk/news)。PyQt5由Riverbank Computing公司开发。

PyQt5是被作为Python的一系列模块使用的。它包括超过620个类以及6000个函数和方法。它是一个跨平台的工具包，可以运行在所有主流的操作系统上，包括Unix，Windows以及Mac OS。PyQt5具有双重许可。开发者可以选择GPL许可或者商业许可。



PyQt5的类分成了如下几个模块：

- **QtCore**

QtCore模块包含除了GUI以外的核心功能。可以用来处理时间、文件以及目录、各种各样的数据类型、流、URL、MIME(Multipurpose Internet Mail Extensions) type、线程或者进程。

- **QtGui**

QtGui包含的类用于窗口化的系统集成、事件处理、2D图形、基本图片、字体以及文本。

- **QtWidgets**

QtWidgets模块包含的类提供了一组UI元素，可以创建经典的桌面风格用户界面。

- **QtMultimedia**

QtMultimedia模块包含处理多媒体内容和连接相机及无线电功能的API类。

- **QtBluetooth**

QtBluetooth模块包含的类可以扫描蓝牙设备并与之连接和交互。

- **QtNetwork**

QtNetwork模块包含的类用来进行网络编程。这些类使得TCP/IP、UDP客户端和服务端的编程更加容易和便捷。

- **QtPositioning**

QtPositioning包含的类使用多种可获得的资源来确定位置，包括卫星、WiFi或文本文件。

- **Enginio**

Enginio模块实现了客户端库，用来管理应用程序托管QT云服务。

- **QtWebSockets**

QtWebSockets模块中的类实现了WebSocket协议。

- **QtWebKit**

QtWebKit包含基于WebKit2库实现的浏览器的类。

- **QtWebKitWidgets**

QtWebKitWidgets包含基于WebKit1实现的浏览器的类，用于QtWidgets基础应用程序。

- **QtXml**

QtXml包含解析XML文件的类。这个模块提供SAX和DOM API的实现。

- **QtSvg**

QtSvg包含显示SVG文件内容的类。Scalable Vector Graphics (SVG) 是一种语言，用XML描述二维图形和图形应用程序

- **QtSql**

QtSql包含有关数据库的类。

- **QtTest**

QtTest包含的**函数**可以对PyQt5应用程序进行单元测试。

# PyQt4和PyQt5的区别

PyQt5并不向下兼容PyQt4，PyQt5有几个重大变化。

但是将旧代码迁移到新版本并不困难，不同的地方如下：

- Python模块已经重写。一些模块已经删除（QtScript），另外一些拆分成了子模块（QtGui，QtWebKit）。
- 引入了一些新的模块，包括 QtBluetooth，QtPositioning，以及Enginio。
- PyQt5 只支持新风格的信号和槽处理机制。对 SIGNAL() 和 SLOT() 的调用不再支持。
- PyQt5不再支持任何 Qt v5.0 中废弃或过时的Qt API。

# Python

Python 是一个通用的动态的面向对象编程语言。Python语言的设计目的强调程序员的生产力和代码的可读性。Python 最初是由 Guido van Rossum 开发的。第一个版本发布于1991年。Python 结合了 ABC, Haskell, Java, Lisp, Icon 以及 Perl 等语言的优点。Python 是一种高级的、通用的、多平台的解释型语言，它是简单的。它最显著的特性是不再使用分号或者括号。它使用缩进代替。目前主要有两个 Python 分支：Python2.x 和 Python 3.x。Python 3.x 与之前的 Python 版本不兼容。它纠正了语言的一些设计缺陷，并使得语言更加简洁。Python 由世界各地的志愿者维护。Python 是一个开源的软件，对于想要学习编程的人来说，Python 是一个理想的选择。

本教程使用 Python 3.x 版本。

Python 支持多种编程风格，它并不强迫程序员某一个特定的模式。Python支持面向对象和面向过程编程，还支持有限的函数式编程。

Python 的官方网站是 [http://python.org/](http://python.org/) 。

Perl、Python 和 Ruby 脚本语言被广泛使用。他们有许多相似的地方并且也是竞争对手。

这一章是对 PyQt5 的介绍。