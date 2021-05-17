---
title: 操作符和递归
date: 2019-04-2 10:19:45
tags: Python
categories: 编程
---

### 向下取整（//）和取余（%）

两个数相除，传统的除法会得到一个浮点数，向下取整会得到一个整数。

<!-- more -->

```Python
>>> minutes = 105
>>> minutes / 60
1.75
>>> minutes // 60
1
>>> minutes % 60
45
```

取余可以用来获取一个数后一位或者后几位数字，例如 x % 10 可以得到个位数字，x % 100 可以得到后两位数字；还可以检验是否能够被一个数整除。

在 Python 2 中，除法操作符（/）左右的操作对象都是整数时，进行的是向下取整操作，而有一个数是浮点数时，进行的就是浮点数操作。

```python
>>> 65 / 3
21
>>> 65.0 / 3
21.666666666666668
```

### 无限递归的避免

函数调用自己也是一种合法行为，我们通常称为递归，但是如果一个递归永远达不到退出条件，就会无限的递归下去。

```python
def recurse():
    recurse()
```

因此写递归函数的时候，一定要加上一个退出的条件，如：

```python
def countdown(n):
	if n <= 0:
		print('Bye!')
	else:
		print(n)
		countdown(n-1)
```

该函数会在 n <= 0 的时候退出。

### 增量开发与函数的简化

通常在编写一个函数的时候会经常遇到输出中间值来验证程序的运行，这些中间值是便于我们调试而产生的，因此这个过程称为增量开发。

例如计算圆的面积：

```python
def circle_area(xc, yc, xp, yp):
    radius = distance(dc, yc, xp, yp)  # 计算圆的半径的函数
    result = area(radius)  # 计算圆的面积
    print(radius, result)
    return result
```

这里面，radius 和 result 都是为了方便调试而设置的中间值，因此可以简化该函数：

```python
def circle_area(xc, yc, xp, yp):
    return area(distance(xc, yc, xp, yp))
```

再比如判断一个数能否被整除：

```python
def is_divisible(x, y):
    if x % y == 0:
        return True
    else:
        return False
```

可以直接简化为：

```python
def is_divisible(x, y):
    return x % y == 0
```

函数代码是不是简洁了许多？当然这种情况只适用于这些值不需要的情况，在大型程序中如果这些值还会用到，就不必简化了，那样反而会使程序太过复杂。