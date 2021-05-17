---
title: 给女朋友讲什么叫接口设计！
date: 2019-03-25 20:43:26
tags: Python
categories: 编程
---

首先介绍一个库，Python 中有一个模块叫 turtle，是一个图形库，可以用来画一些简单的形状。我将基于这个图形库教会大家如何做接口设计。



先来创建一个 turtle 对象。

```python
import turtle
bob = turtle.Turtle()
turtle.mainloop()
```

这段代码就会新建一个窗口，里面包含一个小箭头，这个小箭头就是画图的起点。

![](http://ww1.sinaimg.cn/large/c552abe7ly1g1fccg9qooj20f00db3yd.jpg)

下面介绍一下怎么用 turtle 来画图，以下是常用的方法，记住这些就够了。

```python
import turtle
bob = turtle.Turtle()  # 创建 turtle 对象
bob.fd(100)  # 向前移动 100 个像素
bob.bk(100)  # 向后移动 100 个像素
bob.lt(90)  # 左转 90 度
bob.rt(90)  # 右转 90 度
bob.pu()  # 笔尖朝上
bob.pd()  # 笔尖朝下
turtle.mainloop()
```

### 第一个任务，画一个正方形

当然可以用简单重复的办法来画，事实上新手也是这么做的：

```python
import turtle
bob = turtle.Turtle()

bob.fd(100)

bob.rt(90)
bob.fd(100)

bob.rt(90)
bob.fd(100)

bob.rt(90)
bob.fd(100)

turtle.mainloop()
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1g1fcr2sx4jj20in0crdfp.jpg)

更进一步你想到了用 for 循环：

```python
import turtle
bob = turtle.Turtle()

for i in range(4):
    bob.fd(100)
    bob.rt(90)
    
turtle.mainloop()
```

（细心的同学可能会发现箭头方向变了）

这时，为了提高代码的可读性和重用性，一般会将代码**封装**起来，成为一个函数：

```python
def square(t):
    for i in range(4):
        bob.fd(100)
        bob.rt(90)

square(bob)
```

然后你可能会想了，这正方形的边长都是固定的，我想画一个不一样的怎么办？

### 第二个任务，画一个多边形

这个时候我们需要**泛化**。

```python
def square(t, length):
    for i in range(4):
        bob.fd(length)
        bob.rt(90)

square(bob, 100)
```

更进一步，想要任意边数的多边形：

![](http://ww1.sinaimg.cn/large/c552abe7ly1g1feop1z0oj20fl0f0q2y.jpg)

```python
def polygon(t, n, length):
    angle = 360/n
    for i in range(n):
        bob.fd(length)
        bob.rt(angle)

polygon(bob, 7, 70)
polygon(bob, n=7, length=70)  # 关键字参数
```

现在你就知道了，泛化就是给函数添加参数的过程，当参数比较多的时候可以用关键字参数。

### 第三个任务，画一个圆

这一步要写一个 circle 函数，接受形参 r，表示圆的半径，下面是一个例子：

```python
def circle(t, r):
    circumference = 2 * math.pi * r  # math 是一个数学模块，math.pi 指 π 的值
    n = 50
    length = circumference / n
    polygon(t, n, length)

circle(bob, 70)
```

这个方案的问题在于，n 是一个定值，我们是通过画很多个边，来近似成一个圆的，我们可以加上一个形参，给用户更多的选择，但是，这样**接口**就不太整洁。

函数的接口是如何使用函数的说明：有哪些参数？这个函数做什么？返回值是什么？

说一个接口整洁，是说它能够让调用者完成想要完成的事情，而不用关注细节，这里面 n 就属于画圆的细节，下面是另一个例子：

```python
def circle(t, r):
    circumference = 2 * math.pi * r  # math 是一个数学模块，math.pi 指 π 的值
    n = int(circumference / 3) + 1  
    length = circumference / n
    polygon(t, n, length)
```

这里面 n 的值是一个接近 circumference / 3 的整数，所以每个边长都近似是 3，已经小到足够画出好看的圆了。

![](http://ww1.sinaimg.cn/large/c552abe7gy1g1feq2z78zj20fu0c8mx5.jpg)

### 第四个任务，画一个圆弧

写 circle 函数的时候，可以复用 polygon，因为边数很多的正方形就是圆。但是圆弧则不太好办，因此需要修改 polygon 函数：

```python
def arc(t, r, angle):
    arc_length = 2 * math.pi * r * angle / 360
    n = int(arc_length / 3) + 1
    step_length = arc_length / n
    step_angle = angle / n

    for i in range(n):
        t.fd(step_length)
        t.lt(step_angle)
```

![](http://ww1.sinaimg.cn/large/c552abe7ly1g1fex5thkaj20ey0diglk.jpg)

你会发现，这个函数下面的部分很像 polygon 函数，但是如果不修改 polygon 的接口，就没办法直接用，我们也可以泛化 polygon 函数，但是那样就不叫 polygon（多边形）了，因此将更泛化的名字叫 polyline（多边线）：

```python
def polyline(t, n, length, angle):
    for i in range(n):
        t.fd(length)
        t.rt(angle)
```

现在就可以重写 polygon 和 arc 了：

```python
def polygon(t, n, length):
    angle = 360 / n
    polyline(t, n, length, angle)
    
def arc(t, r, angle):
    arc_length = 2 * math.pi * r * angle / 360
    n = int(arc_length / 3) + 1
    step_length = arc_length / n
    step_angle = angle / n
    polyline(t, n, step_length, step_angle)
```

最后，可以重写 circle，调用 arc：

```python
def circle(t, r):
    arc(t, r, 360)
```

这个过程，叫**重构**。

而一个设计良好的接口，文档字符串是必不可少的，一般用三引号括起来：

```python
def polyline(t, n, length, angle):
    """
    Draws n line segments with the given length and angle between them. t is a turtle.

    """
    for i in range(n):
        t.fd(length)
        t.rt(angle)
```

下面我们总结一下：

我们做一个开发计划一般有以下步骤：

1. 最开始写一些小程序，不需要函数定义；
2. 将部分功能封装成函数；
3. 泛化函数，添加合适的形参；
4. 重复 1 到 3；
5. 重构程序，如果有相似的代码的话，就做成一个更通用的函数。

今天学到了几个概念，一并总结一下：

1. 封装就是将部分功能组成一个函数；
2. 泛化就是给函数添加形参；
3. 接口需要整洁，形参中不要有关于函数内部细节的参数；
4. 重构相似的代码，使代码更为通用。

