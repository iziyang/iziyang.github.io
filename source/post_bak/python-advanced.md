---
title: Python 高级教程
date: 2018-07-09 21:34:54
tags: Python
categories: 编程
---

# 数据结构

## 如何将列表，字典等进行合并

### 列表

- 使用 +
- extend
- 切片

```python
l1 =[1,2,3]
l2 = [4,5,6]
l1 + l2  # 会创建第三个列表
Out[22]: [1, 2, 3, 4, 5, 6]
l1.extend(l2)  # 会修改 l1
l1[len(l1):len(l2)] = l2  # 会修改 l1
```

<!-- more -->

### 字典

```python
# 
context = {}
context.update(defaults)
context.update(user)
# 
context = defaults.copy()
context.update(user)
#
context = dict(defaults)
context.update(user)
# python 3.5
context = {**defaults, **user}
```

## 如何筛选列表、集合、字典中的数据

### 列表

- filter 函数
- 列表解析

列表解析更好，时间更短。

```python
# coding:utf-8

from random import randint

data = [randint(-10, 10) for x in xrange(10)]

# filter 函数
data2 = filter(lambda i: i > 0, data)

# 列表解析
data3 = [x for x in data if x > 0]

print data, data2, data3
```

### 字典

- 字典解析

```python
dictdata = {x: randint(0, 100) for x in xrange(0, 50)}
dictdata2 = {k: v for k, v in dictdata.iteritems() if v >= 60}
```

### 集合

- 集合解析

```python
setdata = set(data)
setdata2 = {x for x in setdata if x > 0}
```

## 如何根据字典中值的大小，对字典中的项排序

- 使用内置函数 sorted
  1. 使用 zip 将字典转换为元组（将值放在第一个）
  2. 使用 sorted 排序
- 使用 sorted 函数的 key 参数

```python
from random import randint
d = {x:randint(60,100) for x in 'xyzabc'}
sorted(zip(d.itervalues(),d.iterkeys()))
# Out[7]: [(65, 'a'), (68, 'x'), (88, 'z'), (91, 'y'), (96, 'b'), (98, 'c')]
sorted(d.iteritems(),key=lambda x:x[1])
```

## 如何为元组中的每个元素命名，提高程序可读性

- 定义类似于其他语言中的枚举类型，也就是定义一系列数值常量
- 使用标准库中的 collections.namedtuple

```python
from collections import namedtuple
student = namedtuple('student',['name','age'])
s1 = student('jim',16)  # 位置参数，也可使用关键字参数
s2 = student(name='tom',age=12)
# 可以使用类对象的方式访问数据
s1.name
# Out[19]: 'jim'
```

## 如何统计序列中元素的出现频度

问题：

1. 找到某随机序列中出现次数最高的三个元素
2. 对某英文文章的单词进行词频统计，找到出现次数最高的 10 个单词

- 构造字典
- 使用 collections 下的 Counter

```python
from random import randint
data = [randint(0, 10) for x in xrange(20)]
# 第一种方式
c = dict.fromkeys(data,0)
for x in data:
    c[x] += 1
# 然后回到之前的问题，根据字典的值，对字典的键排序
sorted(c.iteritems(),key=lambda x:x[1]) 

# 第二种方式
from collections import Counter
c2 = Counter(data)
# c2
# Out[33]: Counter({0: 4, 1: 1, 2: 1, 3: 3, 4: 2, 5: 1, 6: 3, 8: 1, 9: 3, 10: 1})
c2.most_common(3)  # 输出出现频度最高的三个单词
# Out[8]: [(3, 3), (4, 3), (7, 3)]
```

## 如何快速找到多个字典中的公共键

- 使用字典的 viewkeys 方法

```python
from random import randint, sample
s1 = {x: randint(1,4) for x in sample('abcdefg', randint(3,6))}
s2 = {x: randint(1,4) for x in sample('abcdefg', randint(3,6))}
# 取交集
s2.viewkeys() & s1.viewkeys()
# map, reduce
reduce(lambda a, b: a & b, map(dict.viewkeys, [s1, s2]))
```

## 如何让字典保持有序

问题：按照创建元素的顺序创建字典

- 使用 collections 下的 OrderedDict

```python
# 一个简单的竞赛排名系统
from random import randint
from time import time
from collections import OrderedDict

d = OrderedDict()
players = list('ABCDEFGH')
start = time()

for i in xrange(8):
    raw_input()
    p = players.pop(randint(0, 7 - i))
    end = time()
    print i + 1, p, end - start
    d[p] = (i + 1, end - start)
print
print ' ' * 20
for k in d:
    print k, d[k]
```

## 如何实现用户的历史记录功能（最多 n 条）

- 使用 collections 中的 deque（双端循环队列）实现
- 使用 pickle 序列化保存数据

```python
from random import randint
from collections import deque

N = randint(0, 100)
history = deque([], 5)

def guess_digit(k):
    if k == N:
        print 'right'
        return True
    if k < N:
        print '%s is less than N' % k
    else:
        print '%s is greater than N' % k
    return False

while True:
    line = raw_input('please input a number')
    if line.isdigit():
        k = int(line)
        history.append(k)
        if guess_digit(k):
            break
    elif line == 'history':
        print list(history)

# 使用 pickle 序列化对象
import pickle
pickle.dump(history,open('history','w'))
pickle.load('history')
```

# 迭代器与生成器

参考：https://nvie.com/posts/iterators-vs-generators/

![](http://ww1.sinaimg.cn/large/c552abe7gy1ft9cg3hxzvj20zc0fyjrr.jpg)

可迭代对象需要通过 iter () 方法实现为迭代器，然后才能通过 next() 迭代。

生成器是特殊（优雅）的迭代器（通过 yield 方法实现迭代器）。

一句话就是，只有迭代器对象才可以迭代，可迭代对象的意思就是可以实现为迭代器对象，然后迭代，生成器就是迭代器。

## 如何实现可迭代对象和迭代器对象

可迭代对象接口：\_\_iter\_\_()，\_\_getitem\_\_()，__next\_\_()

迭代器对象接口：next()

- 实现一个迭代器对象，next() 方法
- 实现一个可迭代对象，__iter\_\_() 方法返回一个迭代器对象

```python
from collections import Iterable, Iterator
import requests


class WeatherIterator(Iterator):  # 迭代器对象
    def __init__(self, cities):
        self.cities = cities
        self.index = 0

    def get_weather(self, city):
        r = requests.get(u'http://wthrcdn.etouch.cn/weather_mini?city=' + city)
        data = r.json()['data']['forecast'][0]
        return '%s: %s, %s ' % (city, data['low'], data['high'])

    def next(self):
        if self.index == len(self.cities):
            raise StopIteration
        city = self.cities[self.index]
        self.index += 1
        return self.get_weather(city)


class WeatherIterable(Iterable):  # 可迭代对象
    def __init__(self, cities):
        self.cities = cities

    def __iter__(self):
        return WeatherIterator(self.cities)


for x in WeatherIterable([u'北京', u'上海']):
    print x
```

## 如何使用生成器函数实现可迭代对象

- 将 __iter\_\_() 方法实现为 yield() 函数

```python
class PrimeNumbers:  # 生成器实现可迭代对象
    def __init__(self, start, end):
        self.start = start
        self.end = end

    def is_primenum(self, k):
        if k < 2:
            return False
        for i in xrange(2, k):
            if k % i == 0:
                return False
        return True

    def __iter__(self):
        for k in xrange(self.start, self.end + 1):
            if self.is_primenum(k):
                yield k


for x in PrimeNumbers(1, 100):
    print x
```

## 如何进行反向迭代以及如何实现反向迭代

问题：实现一个浮点数发生器，根据范围和步长迭代

- 使用 reversed() 方法

引子：如何将列表反向？

```python
l = [1, 2, 3, 4, 5]
l.reverse()  # 改变了原列表
l[::-1]  # 得到了一个新列表
reversed(l)  # 反向迭代器
iter(l)  # 正向迭代器
```

```python
class FloatFange:
    def __init__(self, start, end, step=0.1):
        self.start = start
        self.end = end
        self.step = step

    def __iter__(self):
        t = self.start
        while t <= self.end:
            yield t
            t += self.step

    def __reversed__(self):
        t = self.end
        while t >= self.start:
            yield t
            t -= self.step


for x in reversed(FloatFange(2, 4)):  # 反向迭代器
    print x
```

## 如何对迭代器做切片操作

问题：有某个文本文件，我们想读取其中某范围的内容，如 100-300 行的内容

- 使用标准库中的 itertools.islice，它能返回一个迭代对象切片的生成器

```python
from itertools import islice

f = open('test')
islice(f, 100, 300)  # 100-300 行
islice(f, 500)  # 前 500 行
islice(f, 100, None)  # 100 行到最后
# islice 会消耗迭代对象。也就是其实是都迭代了，但是只不过把不符合要求的抛弃了
```

## 如何在一个 for 语句中迭代多个可迭代对象

问题：

- 例如有两科成绩，想要算出总成绩
- 例如有三个班的成绩，想要找出大于 90 分的成绩

```python
from random import randint
from itertools import chain

math = [randint(0, 100) for x in xrange(40)]

english = [randint(0, 100) for x in xrange(40)]

for m, e in zip(math, english):  # zip 函数可以迭代多个对象
    sum = m + e
    # print sum

for s in chain(math,english):  # chain 函数可以将多个串联起来
    if s > 90:
        print s
```

# 字符串处理

## 如何拆分含有多种分隔符的字符串

问题：s = 'ab;cd,|enf|def,kdjk;kdjfk\dd' 有多种分隔符，如何处理

- 连续使用 str.split() 方法，每次处理一种分隔符
- 使用正则表达式的 re.split() 方法，一次性拆分字符串

```python
import re

s = 'ab;cd,|enf|def,kdjk;kdjfk sd'


def my_split(s, ds):  # 字符串原始的分隔方法，需要循环
    res = [s]
    print res
    for d in ds:
        t = []
        map(lambda x: t.extend(x.split(d)), res)
        res = t

    return [x for x in res if x]


print my_split(s, ';, |')

split_pattern = '[,; |]'  # 使用正则表达式来分隔
print [x for x in re.split(split_pattern,s) if x]
```

## 如何判断字符串 a 是否以字符串 b 开头或者结尾

问题：某文件目录下有一系列文件，例如 .c，.py，.js

- 使用 str.startwith()，str.endwith()，如果有多个结尾的话，需要用元组传入进去

```python
import os, stat

[name for name in os.listdir('.') if name.endswith(('.sh', '.py'))]

os.stat('e.py').st_mode  # 获取用户权限
stat.S_IXUSR  # 用户权限

os.chmod('e.py', os.stat('e.py').st_mode | stat.S_IXUSR)  # 修改文件权限
```

## 如何调整字符串中文本的格式

问题：例如需要更改 log 文件中的日期表示形式

- 使用正则表达式 re.sub() 方法做字符串替换，利用正则表达式的捕获组，捕获每个部分的内容，在替换字符串中调整各个捕获组的顺序

```python
import re

re.sub('(\d{4})-(\d{2})-(\d{2})', r'\2/\3/\1', file)  # 将某一个日志文件中的日期格式修改，1，2，3 代表捕获组的序号
re.sub('(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})', r'\g<month>/\g<day>/\g<year>', file)  # 第二种方式
```

## 如何将多个小字符串拼接成一个大的字符串

问题：

![](http://ww1.sinaimg.cn/large/c552abe7gy1ft9iukol7tj20qs0ek7di.jpg)

- 迭代列表，使用 + 拼接字符串。（有巨大的开销浪费，拼接过的字符串只是一个临时的）
- 使用 join 方法

```python
s1 = 'ddjfaljfl;'
s2 = 'ddj1222'

s1 + s2  # 运算符重载 + == __add__

l = ['dd', 'kjdlaj', 'kke']
l2 = ''.join(l)
l1 = ['dd', 123, 'dd', 45]
print ['dd', 'kjdlaj', 'kke']
l3 = [str(x) for x in l1]  # 如果列表很长，开销巨大
l3 = (str(x) for x in l1)  # 推荐使用生成器表达式
```

## 如何对字符串进行左，右，居中对齐

问题：

![](http://ww1.sinaimg.cn/large/c552abe7gy1ft9jcq8xbtj20ls0eeadr.jpg)

- 使用字符串的 str.ljust()，str.rjust()，str.center() 对齐
- 使用 format 方法，传递类似 '<20'，'>20'，'^20' 参数

```python
s.ljust(20)
# Out[12]: 'abc                 '
s.rjust(20)
# Out[13]: '                 abc'
s.center(20)
# Out[14]: '        abc         '
format(s,'<20')
# Out[15]: 'abc                 '
format(s,'>20')
# Out[16]: '                 abc'
format(s,'^20')
# Out[17]: '        abc         '

w = max(map(len, d.key()))  # 找出字典中最长的键
for k in d:
    print k.ljust(w), ':', d[k]
```

## 如何去掉字符串中不需要的字符

![](http://ww1.sinaimg.cn/large/c552abe7gy1ft9jodoletj20iq09mgoy.jpg)

- 字符串 strip()，lstrip()，rstrip() 方法取掉字符串两端字符
- 删除单个固定位置的字符，可以使用切片+拼接的方式
- 字符串的 replace() 方法或正则表达式 re.sub() 删除任意位置字符
- 字符串 translate() 方法，可以同时删除多种不同字符

```python
# 去掉字符串两端的字符
s = '   abc    '
s.strip()  # 空白
# Out[19]: 'abc'
s = '---dbc+++'
s.strip('-+')
# Out[21]: 'dbc'

# 去掉固定位置字符
s = 'dbc:ddd'
s[:3] + s[4:]
# Out[23]: 'dbcddd'

# 去掉任意位置
s = '/4kdlkj/4'
s.replace('/4','')
# Out[25]: 'kdlkj'
# 或者用正则表达式的 re.sub

# translate 方法，unicode 也有一个
s = 'abc123456xyz'
import string
string.maketrans('abcxyz','xyzabc')  # 建立映射表
s.translate(string.maketrans('abcxyz','xyzabc'))
# Out[30]: 'xyz123456abc'
s.translate(None,'ax') # 第一个为空，表示删除字符
# Out[33]: 'bc123456yz'

unicode.translate()  # 映射表为一个字典
```



# 文件 I/O 操作

## 如何读写文本文件

关于 python 的编解码，看这一篇就够了：https://foofish.net/why-python-encoding-is-tricky.html

![](http://ww1.sinaimg.cn/large/c552abe7gy1ftaekwcoacj20sy0gmk3l.jpg)

```python
s = u'你好'
# python 2
f = open('filename','w')
f.write(s.encode('utf-8'))  # 写入
f.read().decode('utf-8')  # 读取

# python 3
f = open('filename','rt',encoding='utf-8') # t 指定文本模式
f.write(s)

# tips
# 如果一次性读完了所有文件，需要使用
f.seek(0) # 回到开头
```

## 如何处理二进制文件

二进制文件，例如：视频，图片，录音等。

**大小端字节**

参考：https://blog.csdn.net/hnlyyk/article/details/52541923

大端：低地址存放高字节位

小端：低地址存放低字节位

- open 函数指定模式 'b' 打开文件
- 二进制数据可以用 readinto，读入到提前分配好的 buffer 中
- 解析二进制数据可以使用标准库中的 struct 模块的 unpack 方法

```python
import struct
import array
f = open('demo.wav','rb')
struct.unpack('h','\x01\02')  # 小端存储，h 表示 short
struct.unpack('>h','\x01\02')  # 大端存储
f.seek(0,2)  # 将文件指针移到末尾
f.tell()
n = (f.tell() - 44)/2
buff = array.array('h',(0 for _ in xrange(n)))  # 创建数组
```

## 如何设置文件的缓冲

问题：![](http://ww1.sinaimg.cn/large/c552abe7gy1ftfhmfe084j20oy09a7bg.jpg)

```python
f = open('demo.txt','w',buffering=2048)  # 全缓冲，默认 4096B
f = open('demo.txt','w',buffering=1)  # 行缓冲，遇到换行符就写入一次
f = open('demo.txt','w',buffering=0)  # 无缓冲
```

## 如何将文件映射到内存

![](http://ww1.sinaimg.cn/large/c552abe7gy1ftfhw3rum3j20tq0a8ai2.jpg)

- 使用标准库中的 mmap 模块的 mmap() 函数

```python
import mmap

f = open('demo.bin', 'r+b')
f.fileno()  # 文件描述符
m = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_WRITE)
m[0]  # 索引
m[10:20]  # 切片
m[0] = '\x88'  # 修改文件

#  mmap(fileno, length[, flags[, prot[, access[, offset]]]])

mmap.mmap(f.fileno(), mmap.PAGESIZE * 8, access=mmap.ACCESS_WRITE,offset=mmap.PAGESIZE*4)
```

## 如何访问文件的状态

![](http://ww1.sinaimg.cn/large/c552abe7gy1ftgmrqh1d0j20pq09egri.jpg)

- 系统调用：os, stat 模块
- os.path 下的函数

```python
import os, stat
import os.path

s = os.stat('filename')  # 跟随符号链接，指向原始文件
s
Out[8]: posix.stat_result(st_mode=33188, st_ino=2373061, st_dev=16777217, st_nlink=1, st_uid=501, st_gid=20, st_size=2064, st_atime=1530613571, st_mtime=1530613571, st_ctime=1530613571)

os.lstat('filename')  # 不跟随符号链接
os.fstat('filename')  # 打开的是文件描述符
stat.S_ISDIR(s.st_mode)

# os.path
os.path.isdir()
os.path.islink()
```

## 如何使用临时文件

![](http://ww1.sinaimg.cn/large/c552abe7gy1ftgn6xcmezj20q009i7b9.jpg)

```python
from tempfile import TemporaryFile, NamedTemporaryFile
f = TemporaryFile()  # 在文件系统中找不到
f.write('abcdef'*1000000)
f.seek(0)
f.read(100)
# Out[15]: 'abcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcd'
ntf = NamedTemporaryFile()  # 可以得到临时文件的路径，可以多个进程访问
ntf.name
Out[17]: '/var/folders/6t/b8j3jt1x3nl5bp70ll6fmqfh0000gn/T/tmpa6DRYC'

```

# 数据编码与处理

## 如何读写 csv 数据

- 使用标准库 csv 模块，可以使用其中 reader 和 writer 完成 csv 文件读写

```python
import csv
rf = open('pingan.csv','rb')
reader = csv.reader(rf)
wf = open('pingan.csv','wb')
writer = csv.writer(wf)
writer.writewrow()  # 按行写
```

## 如何读写 json 数据

- 使用标准库中的 json 模块，其中 loads，dumps 函数可以完成 json 数据的读写

```python 
from record import Record  # 录音模块
record = Record(channels=1)
audioData = record.record(2)
```

```python
import json
l = [1, 2, 'abc', {'name': 'bpb', 'age': 'ddd'}]
s = json.dumps(l,separators=[',', ':'])
# Out[9]: '[1,2,"abc",{"age":"ddd","name":"bpb"}]'
json.dumps(d, sort_keys=True)
json.loads(s)
# 文件
json.dump()  # 写入到文件
json.load()  # 从文件导入
```

## 如何解析简单的 xml 文档

- 使用标准库中的 xml 模块

```python
from xml.etree import ElementTree
# 与 xpath 表达式类似
```

## 如何构建 xml 文档

```python
from xml.etree.ElementTree import Element, ElementTree
e = Element('Data')
e.tag
Out[21]: 'Data'
e.set('name', 'abc')
from xml.etree.ElementTree import Element, ElementTree, tostring
tostring(e)
Out[24]: '<Data name="abc" />'
e.text = '123'
tostring(e)
Out[26]: '<Data name="abc">123</Data>'
```

```python
import csv
from xml.etree.ElementTree import Element, ElementTree
from e2 import pretty  # 美化格式的小函数

def csvToXml(fname):
    with open(fname, 'rb') as f:
        reader = csv.reader(f)
        headers = reader.next()

        root = Element('Data')
        for row in reader:
            eRow = Element('Row')
            root.append(eRow)
            for tag, text in zip(headers, row):
                e = Element(tag)
                e.text = text
                eRow.append(e)

    pretty(root)
    return ElementTree(root)

et = csvToXml('pingan.csv')
et.write('pingan.xml')
```

## 如何读写 excel 文件

- 使用 xlrd 和 xlwt（不好用）

# 类与对象

## 如何派生内置不可变类型并修改实例化行为

```python
# coding:utf-8


class IntTuple(tuple):
    def __new__(cls, iterable):
        g = (x for x in iterable if isinstance(x, int) and x > 0)
        return super(IntTuple, cls).__new__(cls, g)

    def __init__(self, iterable):
        # before
        print self
        super(IntTuple, self).__init__(iterable)
        # after


t = IntTuple([1, -1, 'abc', 6, ['x', 'y'], 3])
print t
```

## 如何为创建大量实例节省内存

- 定义类的 __slots\_\_ 属性，用来声明实例属性名字的列表

普通的实例创建完成之后，会有一个 __dict\_\_ 方法，这个方法可以动态绑定实例，但是也造成了很大的内存开销，所以可以通过 \_\_slot\_\_ 属性，阻止动态绑定实例

## 如何让对象支持上下文管理器

- 实现上下文管理协议，需要定义实例的 \_\_enter\_\_, \_\_exit\_\_ 方法，它们分别在 with 开始和结束时被调用

```python
# 在类中实现
def __enter(self):
    #...
    #...
    # 需要返回值 作为 with .. as .. 的对象
    return self

def __exit__(self, exc_type, exc_val, exc_tb):
    # 关闭实例
    return True  # 则不会抛出异常
# 例子
from telnetlib import Telnet
from sys import stdin, stdout
from collections import deque

class TelnetClient(object):
    def __init__(self, addr, port=23):
        self.addr = addr
        self.port = port
        self.tn = None

    def start(self):
        raise Exception('Test')
        # user
        t = self.tn.read_until('login: ')
        stdout.write(t)
        user = stdin.readline()
        self.tn.write(user)

        # password
        t = self.tn.read_until('Password: ')
        if t.startswith(user[:-1]): t = t[len(user) + 1:]
        stdout.write(t)
        self.tn.write(stdin.readline())

        t = self.tn.read_until('$ ')
        stdout.write(t)
        while True:
            uinput = stdin.readline()
            if not uinput:
                break
            self.history.append(uinput)
            self.tn.write(uinput)
            t = self.tn.read_until('$ ')
            stdout.write(t[len(uinput) + 1:])

    def cleanup(self):
        pass

    def __enter__(self):
        self.tn = Telnet(self.addr, self.port)
        self.history = deque()
        return self 

    def __exit__(self, exc_type, exc_val, exc_tb):
        print 'In __exit__'

        self.tn.close()
        self.tn = None
        with open(self.addr + '_history.txt', 'w') as f:
            f.writelines(self.history)
        return True

with TelnetClient('127.0.0.1') as client:
    client.start()

print 'END'

'''
client = TelnetClient('127.0.0.1') 
print '\nstart...'
client.start()
print '\ncleanup'
client.cleanup()
'''

```

## 如何创建可管理的对象属性

问题：不想让用户通过属性来管理，形式上是使用属性访问，实际上调用方法

```python
from math import pi

class Circle(object):
    def __init__(self, radius):
        self.radius = radius

    def getRadius(self):
        return self.radius

    def setRadius(self, value):
        if not isinstance(value, (int, long, float)):
            raise ValueError('wrong type.')
        self.radius = float(value)

    def getArea(self):
        return self.radius ** 2 * pi
    R = property(getRadius, setRadius)  # get 方法，set 方法

c = Circle(3.2)
c.R
c.R = 23
print c.R
```

## 如何让类支持比较操作

- 比较运算符重载，需要实现以下方法：\_\_lt\_\_, \_\_le\_\_, \_\_gt\_\_, \_\_ge\_\_, \_\_eq\_\_, \_\_ne\_\_
- 使用标准库下的 functools 下的类装饰器 total_ordering 可以简化此过程

```python
# 第一种方法，挨个实现
class Rectangle(object):
    def __init__(self, w, h):
        self.w = w
        self.h = h

    def area(self):
        return self.w * self.h

    def __lt__(self, obj):
        print 'in__lt__'
        return self.area() < obj.area()

# 第二种方法
from functools import total_ordering
from abc import abstractmethod


@total_ordering  # 实现运算符重载
class Shape(object):

    @abstractmethod
    def area(self):  # 抽象接口
        pass

    # 而实现其他的大于等于，小于等于等的逻辑都是通过小于和等于来实现的，例如，>= 就是 not __lt__
    def __lt__(self, obj):
        print 'in__lt__'
        if not isinstance(obj, Shape):
            raise TypeError('obj is not shape')
        return self.area() < obj.area()

    def __eq__(self, obj):
        print 'in__eq__'
        if not isinstance(obj, Shape):
            raise TypeError('obj is not shape')
        return self.area() == obj.area()


class Rectangle(Shape):
    def __init__(self, w, h):
        self.w = w
        self.h = h

    def area(self):
        return self.w * self.h


class Circle(Shape):
    def __init__(self, r):
        self.r = r

    def area(self):
        return self.r ** 2 * 3.14


r1 = Rectangle(5, 3)
r2 = Circle(3)

print r1 < r2  #
print r2 < r1
```

## 如何使用描述符对实例属性做类型检查

要求：

- 可以指定类型
- 类型不正确时抛出异常

```python
class Attr(object):
    def __init__(self, name, type_):
        self.name = name
        self.type_ = type_

    def __get__(self, instance, cls):
        return instance.__dict__[self.name]

    def __set__(self, instance, value):
        if not isinstance(value, self.type_):
            raise TypeError('expected an %s' % self.type_)
        instance.__dict__[self.name] = value

    def __delete__(self, instance):
        raise AttributeError("can't delete this attr")

class Person(object):
    name    = Attr('name', str)
    age     = Attr('age', int) 
    height  = Attr('height', float)
    weight  = Attr('weight', float)

s = Person()
s.name = 'Bob'
s.age = 17
s.height = 1.82
s.weight = 52.5
```

## 如何在环状数据结构中管理内存

- 使用标准库 weakref，它可以创建一种能访问对象但不增加引用计数的对象。

```python
import weakref

class Data(object):
    def __init__(self, value, owner):
        self.owner = weakref.ref(owner)  # 弱引用
        self.value = value

    def __str__(self):
        return "%s's data, value is %s" % (self.owner(), self.value)  # 引用函数

    def __del__(self):
        print 'in Data.__del__'

class Node(object):
    def __init__(self, value):
        self.data = Data(value, self)

    def __del__(self):
        print 'in Node.__del__'

node = Node(100)
del node
raw_input('wait...')
```

## 如何通过实例方法名字的字符串调用方法

- 使用内置函数 getattr，通过名字在实例上获取方法对象，然后调用
- 使用标准库 operator 下的 methodcaller 函数调用

```python
# 第一种方法
from lib1 import Cirle
from lib2 import Triangle
from lib3 import Rectangle


def getArea(shape):
    for name in ('area', 'getArea', 'get_area'):
        f = getattr(shape, name, None)
        if f:
            return f()


shape1 = Circle(2)
shape2 = Triangle(3, 4, 5)
shape3 = Rectangle(6, 4)

shapes = [shape1, shape2, shape3]
print map(getArea, shapes)

# 第二种方法
from operator import methodcaller

s = 'abc123abc456'
print s.find('abc', 4)
print methodcaller('find', 'abc', 4)(s)
```

# 多线程与多进程

## 如何使用多线程

- IO 密集型
- CPU 密集型

```python
from threading import Thread


class MyThread(Thread):
    def __init__(self, sid):
        Thread.__init__(self)
        self.sid = sid

    def run(self):
        handle(self.sid)


threads = []
for i in xrange(1, 11):
    t = MyThread(i)
    threads.append(t)
    t.start()
for t in threads:
    t.join()
```

## 如何线程间通信

```python
from Queue import Queue
```

## 如何在线程间进行事件通知

- 等待事件用 wait
- 通知事件用 set

```python
from threading import Event, Thread
```

## 如何使用线程本地数据

## 如何使用线程池

## 如何使用多进程

# 装饰器

## 如何使用函数装饰器

```python
# coding:utf-8

"""
题目 1 计算斐波那契数列 1，1，2，3，5，8，13
"""


def cache(func):
    cache = {}

    def wrap(*args):
        # print args
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]

    return wrap


# 常规写法
@cache  # 加装饰器
def fibonacci(n):
    if n <= 1:
        return 1
    return fibonacci(n - 1) + fibonacci(n - 2)


# 加缓存
def fibonacci_cache(n, cache=None):
    if cache is None:
        cache = {}
    if n in cache:
        return cache[n]
    if n <= 1:
        return 1
    cache[n] = fibonacci_cache(n - 1, cache) + fibonacci_cache(n - 2, cache)
    return cache[n]


# 题目 2 一个共有 10 个台阶的楼梯，从下面走到上面，一次只能迈 1-3 个台阶，并且不能后退，走完这个楼梯共有多少种方法
@cache
def climb(n, steps):
    count = 0
    if n == 0:
        count = 1
    elif n > 0:
        for step in steps:
            count += climb(n - step, steps)
    return count


# 加缓存

# print fibonacci(50)
# print fibonacci_cache(50)

print climb(20, (1, 2,3))
```

## 如何为被装饰的函数保存元数据

![](http://ww1.sinaimg.cn/large/c552abe7gy1ftip0ubmj8j20mg0e2n4j.jpg)

参考：https://foofish.net/python-decorator.html

- functools.wraps

使用装饰器极大地复用了代码，但是他有一个缺点就是原函数的元信息不见了，比如函数的`docstring`、`__name__`、参数列表，先看例子：

```python
# 装饰器
def logged(func):
    def with_logging(*args, **kwargs):
        print func.__name__      # 输出 'with_logging'
        print func.__doc__       # 输出 None
        return func(*args, **kwargs)
    return with_logging

# 函数
@logged
def f(x):
   """does some math"""
   return x + x * x

logged(f)
```

```python
from functools import wraps
def logged(func):
    @wraps(func)
    def with_logging(*args, **kwargs):
        print func.__name__      # 输出 'f'
        print func.__doc__       # 输出 'does some math'
        return func(*args, **kwargs)
    return with_logging

@logged
def f(x):
   """does some math"""
   return x + x * x
```

## 如何定义带参数的装饰器

![](http://ww1.sinaimg.cn/large/c552abe7gy1ftip8g7hnmj20xu0kmk3s.jpg)

### 带参数的装饰器

装饰器还有更大的灵活性，例如带参数的装饰器，在上面的装饰器调用中，该装饰器接收唯一的参数就是执行业务的函数 foo 。装饰器的语法允许我们在调用时，提供其它参数，比如`@decorator(a)`。这样，就为装饰器的编写和使用提供了更大的灵活性。比如，我们可以在装饰器中指定日志的等级，因为不同业务函数可能需要的日志级别是不一样的。

```python
def use_logging(level):
    def decorator(func):
        def wrapper(*args, **kwargs):
            if level == "warn":
                logging.warn("%s is running" % func.__name__)
            elif level == "info":
                logging.info("%s is running" % func.__name__)
            return func(*args)
        return wrapper

    return decorator

@use_logging(level="warn")
def foo(name='foo'):
    print("i am %s" % name)

foo()
```

上面的 use_logging 是允许带参数的装饰器。它实际上是对原有装饰器的一个函数封装，并返回一个装饰器。我们可以将它理解为一个含有参数的闭包。当我 们使用`@use_logging(level="warn")`调用的时候，Python 能够发现这一层的封装，并把参数传递到装饰器的环境中。

`@use_logging(level="warn")`等价于`@decorator`

## 如何实现属性可修改的装饰器

![](http://ww1.sinaimg.cn/large/c552abe7gy1ftipucyfsej20uo0emqd0.jpg)

```python
from functools import wraps
import time, logging
from random import randint


def warn(timeout):
    timeout = [timeout]
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            res = func(*args, **kwargs)
            used = time.time() - start
            if used > timeout[0]:
                msg = '%s, %s, %s' % (func.__name__, used, timeout[0])
                logging.warn(msg)
            return res

        def setTimeout(k):
            #nonlocal timeout  # python 3 语法
            timeout[0] = k

        wrapper.setTimeout = setTimeout  # 将setTimeout()函数作为wrapper的属性

        return wrapper

    return decorator


@warn(1.5)
def test():
    print 'In test'
    while randint(0, 1):
        time.sleep(0.5)


for _ in range(30):
    test()

test.setTimeout(1)
for _ in range(30):
    test()
```

