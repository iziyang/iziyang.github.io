---
title: 一、BeautifulSoup、爬虫异常及浏览器伪装
date: 2018-03-21 14:55:49
tags: [Python,爬虫]
categories: 编程
---

# BeautifulSoup 库

<!-- more -->

```python
# coding:utf-8
from urllib.request import urlopen
from bs4 import BeautifulSoup
from urllib.error import HTTPError
import re

# 基本用法，创建 BeautifulSoup 对象
html = urlopen("http://www.pythonscraping.com/pages/page1.html")
bsObj = BeautifulSoup(html.read(), 'lxml')
print(bsObj.h1)  # 打印标题

# 一个异常处理
def get_title(url):
    try:
        html = urlopen(url)
    except HTTPError as e:
        return None
    try:
        bs_obj = BeautifulSoup(html.read())
        title = bs_obj.body.h1  # 同样可以打印标题
    except AttributeError as e:
        return None
    return title


title = get_title("http://www.pythonscraping.com/pages/page1.html")
if title is None:
    print("Title could not be found")
else:
    print(title)

html = urlopen("http://www.pythonscraping.com/pages/warandpeace.html")
bsObj = BeautifulSoup(html.read(), 'lxml')
print(bsObj.h1)
nameList = bsObj.findAll("span", {"class": "green"})  # 找到 span 标签，属性为 class=green 的内容
for name in nameList:
    print(name)
    print(name.get_text())  # 获取到文本内容

html = urlopen("http://www.pythonscraping.com/pages/page3.html")  # 处理子标签
bsObj = BeautifulSoup(html, 'lxml')
for child in bsObj.find("table", {"id": "giftList"}).children:  # 子标签
    print(child)

html = urlopen("http://www.pythonscraping.com/pages/page3.html")  # 处理兄弟标签
bsObj = BeautifulSoup(html)
for sibling in bsObj.find("table", {"id": "giftList"}).tr.next_siblings: # 下面的兄弟标签
    print(sibling)

html = urlopen("http://www.pythonscraping.com/pages/page3.html")  # 处理父标签
bsObj = BeautifulSoup(html, 'lxml')
print(bsObj.find("img", {"src": "../img/gifts/img1.jpg"
                         }).parent.previous_sibling.get_text())  # 父标签的前一个标签

html = urlopen("http://www.pythonscraping.com/pages/page3.html")
bsObj = BeautifulSoup(html, 'lxml')
images = bsObj.findAll("img", {"src": re.compile("\.\.\/img\/gifts/img.*\.jpg")})  # 可以同时使用正则表达式
for image in images:
    print(image["src"])  # 获取属性

print()

images = bsObj.findAll("img")  # 取属性
for image in images:
    print(image.attrs['src'])  # 同样可以获取属性

```

# 爬虫的异常处理

## 异常处理

爬虫中的异常分为两种，一种是 HTTPError，另一种是 URLError，URLError 是 HTTPError 的父类，URLError不包括异常的状态码及原因而 HTTPError 包括，因此使用以下的方法来输出异常状态码和原因：

```python
import urllib.error
import urllib.request

try:
    urllib.request.urlopen("http://blog.csdn.net")
except urllib.error.URLError  as e:
    if hasattr(e,"code"):
        print(e.code)
    if hasattr(e,"reason"):
        print(e.reason)
```

## 异常原因

产生 URLError 的原因有以下几种：

1. 连不上服务器
2. 远程 URL 不存在
3. 本地没有网络
4. 触发 HTTPError 子类

## 常见异常

301：重定向到新的 URL，永久性

302：非永久性的重定向

304：请求资源未更新

400：非法请求

401：请求未授权

403：禁止访问

404：找不到页面

500：服务器内部出错

501：服务器不支持实现请求所需要的功能

# 浏览器伪装技术

```python
import urllib.request

url="http://blog.csdn.net/weiwei_pig/article/details/52123738"
# 添加浏览器的 User-Agent
headers=("User-Agent","Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.22 Safari/537.36 SE 2.X MetaSr 1.0")
opener=urllib.request.build_opener()  # 创建对象
opener.addheaders=[headers]  # 添加浏览器头

data=opener.open(url).read()
fh=open("F:/天善-Python数据分析与挖掘课程/result/22/4.html","wb")
fh.write(data)
fh.close()
```

