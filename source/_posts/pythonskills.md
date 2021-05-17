---
title: 如何使 Python 更简洁
date: 2019-04-19 09:33:02
tags: Python
categories: 编程
---

#### 条件表达式

我们经常会使用条件语句来从两个值中选择一个，例如：

```python
if x > 0:
    y = math.log(x)
else:
    y = float('nan')
```

这几行代码会检查 x 是否为正数。如果为正数，则计算 math.log；如果为负数，math.log 会抛出 ValueError 异常。为了避免程序停止，则直接生成了“NAN”，一个特殊的浮点数，代表“不是数”。

实际上这几行代码可以使用更加简洁的方式：

```python
y = math.log(x) if x > 0 else float('nan')
```

这条语句是不是可以非常简洁？

有时候，递归表达式也可以用条件表达式重写，以斐波那契数列为例：

```python
def factorial(n):
    if n == 0:
        return 1
    else:
        return n * factorial(n-1)
```

可以将它重写为：

```python
def factorial(n):
    return 1 if n ==0 else n * factorial(n-1)
```

一般来说，如果条件语句的两个分支都只包含简单的判断或对同一变量进行赋值的表达式，那么就可以转化为条件表达式。

#### 列表解析式和生成器

列表解析式可以快速生成一个列表，例如假如需要对列表中的元素做某些操作，生成一个新的列表，可能我们会这样写：

```python
def capitalize_all(t):
    res = []
    for s in t:
        res.append(s.capitalize())
    return res
```

而使用列表解析式操作是这样子的：

```python
def capitalize_all(t):
    return [s.capitalize() for s in t]
```

生成器表达式与列表解析式有一点不同，那就是生成器表达式使用圆括号：

```python
>>> g = (x**2 for x in range(5))
>>> g
<generator object <genexpr> at 0x000001B46BB8FE08>
```

这个结果是一个生成器对象，它不会把所有的结果一次性计算出来，而是请求一个获取一个，它使用 next 函数获取下一个值：

```python
>>> next(g)
0
>>> next(g)
1
```

生成器表达式经常和 sum、max 和 min 之类的函数配合使用：

```python
>>> sum(x**2 for x in range(5))
30
```

#### any 和 all

Python 中有两个内置函数 any 和 all。any 接收一个由布尔值组成的序列，并当其中任何值是 True 时返回 True。它可以用于列表：

```python
>>> any([True, False])
True
```

但更常见于生成器表达式：

```python
>>> any(letter == 't' for letter in 'monty')
True
```

all 函数则是在序列中所有元素都是 True 时返回 True。

#### 集合和 collections 模块

集合 set 是 Python 中的另一种数据类型，集合中的元素是没有重复的。

例如，判断一个字符集是不是在另一个字符集中全部出现：

```python
def uses_only(word, available):
    return set(word) <= set(available)
```

操作符 <= 检查一个集合是否是另一个集合的子集。

collections 模块包含了一些常用的方法，例如计数器 Counter、defaultdict、命名元组等。