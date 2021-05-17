---
title: Linux 基础知识
date: 2018-05-17 09:42:35
tags: Linux
categories: 编程
---

## Linux 文件属性代表

```shell
[root@localhost ~]# ll
total 171164
-rw-------. 1 root root      1261 Apr 27 06:50 anaconda-ks.cfg
-rw-r--r--. 1 root root      3624 Apr 28 21:50 install.txt
-rw-r--r--. 1 root root 175262413 Apr  3 14:05 jdk-8u171-linux-x64.rpm
drwxr-xr-x. 3 root root       132 May 13 21:47 script
```



-：代表普通文件

d：代表目录

l：代表链接文件

b：代表块设备文件

c：代表字符设备，如鼠标等

## Linux 命令

### 开关机

```shell
shutdown -r now # 重启
init 0 # 关机
init 3 # 纯命令行
init 5 # 含有图形界面
init 6 # 重启
```

### 权限

```shell
chgrp # 改变文件所属用户组
chown # 改变文件所有者 可以用:或.分割所有者与用户组来进行改变，如 root:root
chmod # 改变文件权限 -R 递归改变
```

### 目录

```shell
pwd -P # 列出链接文件的完整文件名
mkdir
	-p # 多级创建目录
	-m # 给新建目录添加权限 mkdir -m 777 test
cp -s -l -a # 软链接，硬链接，所有属性
ls -l --time= # atime：权限或属性改变的时间
			  # ctime：读取文件内容的时间
```

### 查看文件

```shell
cat
	-n # 打印出行号
tac # 从最后一行开始显示
nl # 输出行号
more # 一页一页的显示
	 # 空格键：向下翻一页
	 # enter：向下滚动一行
	 # /：向下查询
	 # :f：立刻显示出文件名以及目前显示的行数
	 # b：往前翻页，对管道不行
less # 与 more 类似，但是可以向前翻页
	 # 可以向前/或者向后?查询字符串
head # 显示头几行
tail # 显示后几行
	 # 这两个都是默认显示 10 行，-n 参数指定行数
od # 以二进制方式读取
	-t # 后面可以指定文件类型。a：默认字符；c：ascii字符；d：十进制
touch [-acdmt] # 可以修改文件时间
```

### 权限管理

```shell
umask # 默认权限管理，文件最大权限666，目录777
chattr +i +a # 设置默认属性。i：设置文件无法被改动／删除／设置链接等；a：只能增加数据，不能删除修改数据
lsattr # -a：列出隐藏属性
SUID # 使得用户暂时获得执行者权限，4
SGID # 使得用户暂时获得用户组权限，2
SBIT # 其他人无法修改别人创建的文件，只能修改自己的文件，1
```

### 文件类型

```shell
file # 可以查看文件类型
```

### 文件查找

```shell
which # 查找命令
whereis # 查找文件
locate # 可以按照部分文件名查找文件
	updatedb # 更新数据库
```

### 解压缩命令

```
tar -zcvf filename.tar.gz 要压缩的文档
tar -zxvf filename.tar.gz -C 目录

	-z 表示.gz
	-j 表示.bz2
	-c|x 表示压缩/解压
	-v 表示显示文件名
	-f 表示指定文件名
```



## shell

```shell
type # 可以显示命令类型，属于系统自带还是外部
unset 变量名 # 取消变量设置
单引号会让引号内的内容失去转义，而双引号则不会。
引号内如果需要执行命令，则可以采用 $(命令)的形式，或者采用反单引号
locale -a # 可以显示当前系统支持的语系
read variable # 读取输入内容
declare # 声明变量类型
	-a：# 定义为数组
	-i：# 定义成为整数数字
	-x：# 将变量变成环境变量
	-r: # 将变量设置成为 readonly
	
# 代表删除最短变量
## 代表删除最长变量
这两个是从前向后删除
% %% # 从后向前
/ // # 替换字符串
变量测试与内容替换
- 与 = ？
历史命令
!number # 执行第几条命令
!command # 执行上一个以 command 开头的命令
/etc/issue /etc/motd # 可以设置欢迎信息与提示信息
ctrl + c # 终止命令
ctrl + z # 暂停命令
```

#### 输出重定向

```Shell
1>
2>
将错误信息与正确信息分别输入到不同的文件中去
find /home -name .bashrc > list_right 2> list_error
将正确与错误信息统统写入同一个文件
find /home -name .bashrc > list 2>&1
tee # 双向重定向
	-a 表示累加
```

#### 命令的一次执行

```
; # 表示下一个命令与上一个命令没有关系
&& # 表示上一个命令执行正确之后才会执行下一个命令（与）
|| # 表示上一个命令执行错误才执行下一个命令（或）
```

### 选取命令

```Shell
cut -d '分隔符' -f 分隔后的区域块
cut -c 字符范围 # 用于排列整齐的信息
last # 可以显示登录者信息
```

### 排序命令

```shell
sort # 排序
	-n # 使用纯数字排序
	-t # 分隔符
	-k # 区间 
uniq # 将重复数据只显示一次
	-c # 计数
wc -l -w -m # 行数／字数／字符数
```

### 字符转换命令

```shell
tr str1 str2 # 将 1 替换为 2 
	-d # 删除字符串
	-s # 替换掉重复的字符串
col # 字符转换命令
	-x # 将 tab 键转换成对等的空格键
join # 将有相同字段的连接为一行
	-t # 默认以空格符分隔，并且对比第一个字段的内容
paste # 直接将两行贴在一起
expand # 将tab转换为空格 -t 接数字，表示空格键个数
split # 切割命令
	-b # 按照文件大小切割
	-t # 按照行数切割
xargs # 参数代换，可以将标准输出作为某一个命令的参数
- # 表示标准输入或输出
sed -n # 安静模式
	a # 新增
	c # 替换
	d # 删除
	i # 插入
	p # 打印
	s # 替换
sed 's/str1/new_str/g'
```

### 测试及判断命令

```shell
test
	-e # 该文件名是否存在
	-f # 该文件名是否存在且是否为文件
	-d # 目录
	-rwx # 文件权限
	还可以测试文件之间的比较，整数之间的判定，判定字符串的数据等
	-ne # 两数值不等
	-eq # 两数值相等
	-gt # 大于
	-lt # 小于
	-ge #
	-le #
	-n # 判断字符串是否为非空
	-z # 判断字符串是否为空
	-a # 与
	-o # 或
	! # 非
```

### shell 参数

```shell
$0 # 脚本名字
${1..n} # 第几个参数
$# # 参数个数
$@ # 代表"$1"/"$2"...
shift # 变量的偏移，后面可以接数字，也即删除掉前面的几个参数
```

### 条件判断

```Shell
if [];then
...
fi

if [];then
...
elif [];then
...
else
...
fi

case $variable in
	"name")
	...
	;;
	"")
	...
	;;
	*)
	exit 1
	;;
esac
```

### 函数

```shell
function fname ()
	{...}
	
需要注意，函数中的 $0,$1,$2,与脚本中的参数不同，这个参数指的是函数的形参
```

### 循环

```shell
while []
do
	...
done

until []
do
	...
done

for var in con1 con2 con3
do
	...
done

$(seq 1 100) # 1 到 100

for ((初始值;限制值;步长))
do
done
```

### 语法检查

```
bash -nvx 
	-n # 查询语法问题
	-v # 执行脚本之前，现将内容输出
	-x # 将使用到的脚本内容显示到屏幕上
```

## 系统相关

### 后台进程

```shell
& # 代表把程序放到后台执行
jobs # 可以查看后台程序，默认只显示jobnumber
	-l # 列出 PID 号码
fg %jobnumber # 把程序拿到前台处理
fg - # 把状态为 - 的拿到前台，默认+
bg %jobnumber # 把后台任务状态变为 running
```

### 系统资源查看

```shell
free # 查看内存使用情况
uname # 查看系统与内核相关信息
uptime # 查看系统启动时间与工作负载
```

### 查询已打开文件或已执行程序打开的文件

```shell
fuser file／dir # 
lsof # 显示进程所打开的文件
pidof # 找出某个正在执行的进程的 PID
```

