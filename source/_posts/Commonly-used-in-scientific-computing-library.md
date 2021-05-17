---
title: 常用的科学计算库
date: 2018-01-17 15:14:36
tags: Python
categories: 编程
---

# matplotlib 模块

matplotlib 是一个数据可视化函数库 ，matplotlib 的子模块 pyplot 提供了 2D 图表制作的基本函数 ，官方网站上有许多例子可供参考：

[https://matplotlib.org/gallery.html ](https://matplotlib.org/gallery.html )

<!--more-->

## 散点图绘制

```python
import matplotlib.pyplot as plt
plt.scatter(x, y) # x, y 分别是 x 坐标和 y 坐标的列表
plt.show()
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1fnjmt56wmvj20a7085dgg.jpg)

## 直方图

```python
import matplotlib.pyplot as plt
plt.hist(data, bins)
```

data：数据列表
bins:：分组边界

![](http://ww1.sinaimg.cn/large/c552abe7ly1fnjmwnflkvj20gl02r755.jpg)

![](http://ww1.sinaimg.cn/large/c552abe7ly1fnjmwwjvklj20c7099t8o.jpg)

## 标签设置

```
plt.xticks()  # 设置 x 坐标的坐标点位置及标签
plt.title()  # 设置绘图标题
plt.xlabel(), plt.ylabel()  # 设置坐标轴的标签
```

代码参考：F:\编程语言\python\零基础python入门\零基础Python入门课件和代码\lect08_代码\代码

# NumPy

NumPy (Numeric Python)：用 Python 实现的科学计算库

包括：

1. 强大的 N 维数组对象 array
2. 成熟的科学函数库
3. 实用的线性代数、随机数生成函数等
4. NumPy 的操作对象是多维数组 ndarray
  - ndarray.shape 数组的维度
  - 创建数组：np.array(<list>)，np.arrange() …
  - 改变数组形状  reshape() 

## 创建随机数组

```
np.random.randint(a, b, size)  # 创建 [a, b) 间形状为 size 的数组 
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1fnjn8tah9rj20ep080gme.jpg)

## NumPy 基本运算

以数组为对象进行基本运算，即向量化操作。

![](http://ww1.sinaimg.cn/large/c552abe7ly1fnjoykhwiuj20cl03wt92.jpg)

np.histogram() 输出直方图的统计结果。