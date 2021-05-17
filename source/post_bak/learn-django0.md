---
title: 1. 如何搭建一个虚拟的 Python 环境？
date: 2018-11-07 14:44:27
tags: [从零开始学 Django,Python]
categories: 编程
---

通常在我们在编写 Python 程序的时候，需要什么库我们就直接用 pip 安装了，这种情况下是全局式的安装。我们在创建一个 Python 项目的时候，例如有的项目需要 Python3，有的需要 Python2，为了避免项目依赖的互相影响以及去除不需要的库，通常会通过虚拟环境来解决。

这个虚拟环境其实相当于另外独立出一个 Python 环境，有自己的独立的库，独立的依赖关系，是在开发项目时候的利器。话不多说，下面讲一下使用方法。

<!--more-->

# 使用 virtualenv

virtualenv 是一个 Python 库，通过它我们可以很方便的虚拟出一个 Python 环境。

1. 首先安装 virtualenv 

   ```shell
   pip install virtualenv 
   ```

2. 创建虚拟环境

   ```shell
   virtualenv env
   ```

   没错，就是这么简单！这里 env 是你所要创建的虚拟环境的目录。

   创建后会有如下几个文件：

   ```powershell
   2017/12/15  17:16    <DIR>          Include
   2018/11/08  10:25    <DIR>          Lib
   2018/11/08  10:25    <DIR>          Scripts
   2018/11/08  10:25    <DIR>          tcl
                  0 个文件              0 字节
                  6 个目录 44,746,711,040 可用字节
   ```

3. 激活虚拟环境

   首先进入 env/Scripts 目录：

   ```powershell
   2018/11/08  10:25             2,173 activate
   2018/11/08  10:25               754 activate.bat
   2018/11/08  10:25             8,325 activate.ps1
   2018/11/08  10:25             1,143 activate_this.py
   2018/11/08  10:25               512 deactivate.bat
   2018/11/08  10:25           102,777 easy_install-3.6.exe
   2018/11/08  10:25           102,777 easy_install.exe
   2018/11/08  10:25           102,759 pip.exe
   2018/11/08  10:25           102,759 pip3.6.exe
   2018/11/08  10:25           102,759 pip3.exe
   2018/11/08  10:25           100,504 python.exe
   2018/11/08  10:25            58,520 python3.dll
   2018/11/08  10:25         3,610,776 python36.dll
   2018/11/08  10:25            98,968 pythonw.exe
   2018/11/08  10:25           102,755 wheel.exe
                 15 个文件      4,498,261 字节
                  2 个目录 44,732,145,664 可用字节
   ```

   执行 activate 就激活了虚拟环境：

   ```powershell
   C:\Users\北子\env\Scripts>activate
   (env) C:\Users\北子\env\Scripts>
   ```

   激活之后会看到在命令行前面会加上一个括号，表明当前的虚拟环境。

4. 取消激活虚拟环境

   取消激活也很简单，只需要执行 deactivate 即可：

   ```powershell
   (env) C:\Users\北子\env\Scripts>deactivate.bat
   C:\Users\北子\env\Scripts>
   ```

5. virtualenv 相关命令

   这里只列举了最常用的命令：

   ```shell
   Options:
     --version             show program's version number and exit
     -h, --help            show this help message and exit
     -p PYTHON_EXE, --python=PYTHON_EXE  # 可以指定 Python 版本，不指定的话则基于全局版本
                           The Python interpreter to use, e.g.,
                           --python=python3.5 will use the python3.5 interpreter
                           to create the new environment.  The default is the
                           interpreter that virtualenv was installed with
                           (c:\python36\python.exe)
     --no-site-packages    # 目前是默认的。即安装的时候不会安装全局的包。
     --no-setuptools       Do not install setuptools in the new virtualenv.
     --no-pip              Do not install pip in the new virtualenv.
     --no-wheel            Do not install wheel in the new virtualenv.
   ```

虚拟环境中安装包的话，只需要激活对应的虚拟环境，使用 pip 安装即可。

细心的同学可能会发现这里有一个问题，那就是每次创建虚拟环境都需要进入 Scripts 目录执行激活和取消激活的命令，而且 virtualenv 还不能查看当前有哪些虚拟环境，这时候就需要用到 virtualenvwrapper 咯！

## 用 virtualenvwrapper 管理虚拟环境

### 安装

在 Linux 上直接用 pip 安装就好：

```shell
$ pip install virtualenvwrapper
```

Windows  上安装 virtualenvwrapper-win：

```powershell
pip install virtualenvwrapper-win
```

### 设置 WORKON_HOME 变量

这个变量主要是用来指定虚拟环境的存放位置的，如不指定则会在默认位置。

**Linux**：

```shell
export WORKON_HOME=$HOME/.virtualenvs
```

$HOME/.virtualenvs 这是默认位置，可以修改。

**Windows**：

默认位置 %USERPROFILE%\Envs。

需要在环境变量中增加 WORKON_HOME 变量。

![](http://ww1.sinaimg.cn/large/c552abe7gy1fx0i2dqpk1j20nb061aa2.jpg)

### 使用

使用 virtualenvwrapper 非常方便，下面列举一下常用的命令。

```shell
mkvirtualenv <name>  # 创建虚拟环境,这里也可用 virtualenv 的相关选项
lsvirtualenv  # 列出 WORKON_HOME 目录下所有的虚拟环境
rmvirtualenv <name>  # 删除虚拟环境
workon [<name>]  # 激活虚拟环境，类似于 activate
deactivate  # 取消激活
```

还有其他的一些命令就不再一一列举了，感兴趣的可以去看看官方文档。

# 使用 conda

使用 conda 管理虚拟环境需要先安装 Anaconda。Anaconda 是一个开源的 Python 版本，其特点在于包含了一大堆科学计包，感兴趣的朋友可以去了解，这里不再赘述。使用 conda 命令行工具也可以创建虚拟环境。

1. 创建虚拟环境

   ```shell
   conda create -n your_env_name python=X.X
   ```

   创建 Python 版本为 X.X、名字为your_env_name的虚拟环境。your_env_name 文件可以在 Anaconda 安装目录 envs 文件下找到，如不指定 Python 版本，则会基于 Anaconda 的版本创建。

2. 激活虚拟环境

   ```shell
   source activate your_env_name  # Linux
   activate your_env_name  # Windows
   ```

3. 虚拟环境中安装包

   ```shell
   conda install -n your_env_name [package]
   ```

4. 退出虚拟环境

   ```shell
   source deactivate  # Linux
   deactivate  # Windows
   ```

5. 删除虚拟环境。

   ```shell
   conda remove -n your_env_name --all
   ```

6. 删除环境中的某个包。

   ```shell
   conda remove --name your_env_name  package_name
   ```

## conda 常用的命令

```shell
conda -V  # 检验是否安装以及当前 conda 版本
conda list  # 查看安装了哪些包。
conda env list 或 conda info -e  # 查看当前存在哪些虚拟环境
conda update conda  # 检查更新当前conda
conda -h  # 查看 conda 帮助
```

# 项目部署

当采用以上方法建立一个项目之后，在部署的时候该如何操作呢？

## virtualenv

首先将虚拟环境中的包导出：

```shell
pip freeze > plist.txt
```

其次在生产环境安装包：

```shell
pip install -r plist.txt
```

## conda

导出环境：

```shell
conda env export > environment.yaml  # 将包保存为 YAML
```

部署环境：

```shell
conda env create -f environment.yaml  # 通过 YAML 文件安装包
```

# 总结

如果没有大量的科学计算的话，个人建议还是使用 virtualenv 比较好，因为 virtualenv 会使你的 Python 环境最为干净。

参考：

https://blog.csdn.net/lyy14011305/article/details/59500819 

https://virtualenvwrapper.readthedocs.io/en/latest/install.html

https://virtualenv.pypa.io/en/latest/

https://pypi.org/project/virtualenvwrapper-win/