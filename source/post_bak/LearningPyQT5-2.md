---
title: 第一个 PyQt5 程序
date: 2017-06-08 13:37:08
tags: [Python,PyQT]
categories: 编程
---

# 第一个PyQt5程序

> 这一节我们学习一些基本的操作。

## 简单的例子

这是一个简单的例子，用来显示一个小窗口。但是我们可以在这个窗口上做很多东西，我们可以调整大小、最大化或者最小化它，这需要很多代码，不过已经有人编好了代码。由于窗口在大多数应用中都要重复使用，所以就没有必要再重复编码了。PyQt5是一个高级的工具，如果我们使用低级的工具包编码，下面的示例代码很容易就达到几百行。
<!-- more -->

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-

"""
ZetCode PyQt5 tutorial 

In this example, we create a simple
window in PyQt5.

author: Jan Bodnar
website: zetcode.com 
last edited: January 2015
"""

import sys
from PyQt5.QtWidgets import QApplication, QWidget


if __name__ == '__main__':
    
    app = QApplication(sys.argv)

    w = QWidget()
    w.resize(250, 150)
    w.move(300, 300)
    w.setWindowTitle('Simple')
    w.show()
    
    sys.exit(app.exec_())
```

上面的代码会在屏幕上显示一个小窗口。

```python
import sys
from PyQt5.QtWidgets import QApplication, QWidget
```

这里我们进行了必要的导入。基本部件在 **PyQt5.QtWidgets** 模块中。

