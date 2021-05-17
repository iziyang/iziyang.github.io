---
title: 文本处理命令
date: 2018-05-30 17:33:07
tags: Linux
categories: 编程
---

一些小命令：

```shell
grep --color=auto ''
grep -n 'str' file # 查找 str -n 表示行号
grep -vn 'str' file # 查找不包含 str 的字符串
grep -in 'str' file # 不论大小写
```



# 正则

```shell
^ # 行首
$ # 行尾
. # 表示任意
\w # 字类字符
\W # 非字类字符
\b # 单词分隔
* # 0 个或多个
\{n,m\} # 连续 n 到 m 个前一个字符
+ # 一次或多次
? # 零次或一次
| # 或
```

# sed

```shell
[root@www ~]# sed [-nefr] [动作]
参数:
    -n # 使用安静(silent)模式。
    -e # 直接在指令列模式上进行 sed 的动作编辑
    -f # 直接将 sed 的动作写在一个文件内，-f filename
    -r # sed 支持延伸型正则表示法
    -i # 直接修改读取的文件内容
动作说明: [n1[,n2]]function
    n1, n2 # 行数
function:
    a # 新增，在后面
    c # 取代，取代 n1,n2 之间的行数
    d # 删除
    i # 插入，在前面
    p # 打印，通常和 sed -n 一起
    s # 取代，例如 1,20s/old/new/g
```



```shell
sed '2,5d'
sed '2a drink tea'
sed '2a Drink tea or ......\
> drink beer ?'
sed '/^$/d' # 删除空白行
sed 's/#.*$//g' # 删除注释行
sed -i 's/\.$/\!/g' regular_express.txt # 直接修改文件内容，将 . 替换为 !
```

# awk

```shell
awk [options] 'command' file
    command:pattern{命令}
    $0 # 当前行
    $1 # 第一个字段，以此类推
    -F ':' # 分隔符
    NR # 每行记录号
    NF # 字段数
    逻辑运算符 > < >= <= == !=
    逻辑判断 if...else...
    正则表达式 ~ !~
awk -F ':' '$1~/^m.*/{print $1}' file # 表示 $1 以 m 开头
扩展格式：
	BEGIN{}pattern{}END{}
awk -F ":" 'BEGIN{print 'line col user'}{print NR,NF,$1}END{print ...}'
```

