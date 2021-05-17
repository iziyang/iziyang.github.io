---
title: 用 Hexo 在 Github 上面部署博客
date: 2017-03-11 15:28:30
tags: [hexo,blog]
categories: 编程
---
# 一、环境安装 #
1. 安装 Node.js [https://nodejs.org/en/](https://nodejs.org/en/)
2. 安装 Git [https://github.com/waylau/git-for-win](https://github.com/waylau/git-for-win)

<!-- more -->
3. 安装完成后，在开始菜单里找到“Git”->“Git Bash”，名称和邮箱是Github上的。
  `git config --global user.name "***"`
  `git config --global user.email "i**@**.com"`

4. 安装 Hexo。所有必备的应用程序安装完成后，即可使用 npm 安装 Hexo。在命令提示符中：`npm install -g hexo-cli`好了到这一步我们环境全部安装好了。

# 二、设置 #


1. 在命令行提示符中进入要设置blog的目录`hexo init blog`这条命令会创建一个blog的文件夹。提示`INFO  Start blogging with Hexo!`
2. 初始化hexo 之后source目录下自带一篇hello world文章, 所以直接执行下方命令：
   `$ hexo generate #启动本地服务器`
   `$ hexo server #在浏览器输入 http://localhost:4000/就可以看见网页和模板了`
   `INFO  Start processing`
   `INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.`
   这里发现网页一直无响应，应该是4000端口被占用了，所以这里换成5000
   `hexo server -p 5000`
   成功！
   ![alt text](/images/1.png)

3.将ssh-keygen.ext添加到环境变量，一般在git安装文件夹下bin下面。重新打开**git bash**，输入：`ssh-keygen -t rsa -C "Github的注册邮箱地址"`



> 之前一直在命令行提示符中执行命令，结果提示找不到`ssh-keygen`

一路Enter过来就好，得到信息：
​    
    Your public key has been saved in /c/Users/user/.ssh/id_rsa.pub.

找到该文件，打开（sublime text），`Ctrl+a`复制里面的所有内容，然后进入GitHub：[https://github.com/settings/ssh](https://github.com/settings/ssh)

> New SSH key ——Title：blog —— Key：输入刚才复制的—— Add SSH key

# 三、配置博客 #
在blog目录下，用sublime（为了防止乱码，yml是utf-8编码）打开_config.yml文件，修改参数信息

特别提醒，在每个参数的：后都要加一个空格

修改网站相关信息

    title: 北子说
    subtitle: Some essays
    description: 我还是喜欢你，像风走了八万里，不问归期。
    author: 北子
    language: zh-Hans
    timezone: Asia/Shanghai

配置部署

    deploy: 
      type: git
      repo: https://github.com/yuanzizhen/yuanzizhen.github.io.git
      branch: master
# 四、发表文章 #
在命令行提示符中输入

    $ hexo new "北子测试文章"
    INFO  Created: F:\test\blog\source\_posts\北子测试文章.md
找到该文章，打开，使用Markdown语法

    ---
    title: 北子测试文章
    date: 2017-02-28 13:03:44
    tags:
    ---
    这是一篇测试文章，欢迎关注作者博客: https://yuanzizhen.github.io/
保存，然后执行下列步骤：

    $ hexo clean
    $ hexo generate
    $ hexo server -p 5000
这个时候，打开http://localhost:5000/，发现看不到刚刚编辑的文章。原因：命令行提示符中执行

    npm install hexo-deployer-git --save
成功！

![alt text](/images/2.png)
> 这里乱码的原因是，配置文件中的中文没有用utf-8编码，因此，在配置的时候一定要注意编码。

最后一步，发布到网上，执行：

    $ hexo deploy
其中会跳出Github登录，直接登录，如果没有问题输入yuanzizhen（换成你的）.github.io/,然后就可以看到已经发布了

# 五、修改主题 #
我这里使用next主题，GitHub上面star数最多的主题。
[https://github.com/iissnan/hexo-theme-next](https://github.com/iissnan/hexo-theme-next) 而且还有详细的安装文档。
这里不再赘述。另外，我还添加了第三方服务多说评论，多说分享等。

在修改菜单项的时候遇到了不少错误，最终的配置如下：

在`主题配置文件`中

    menu:
      home: /
      #categories: /categories
      #about: /about
      archives: /archives
      tags: /tags
      #sitemap: /sitemap.xml
      #commonweal: /404.html
      openstack: /categories/云计算
      code&tech: /categories/code-tech
      book: /categories/颜如玉
      essay: /categories/杂谈

同时修改图标：

    menu_icons:
      enable: true
      #KeyMapsToMenuItemKey: NameOfTheIconFromFontAwesome
      home: home
      about: user
      categories: th
      schedule: calendar
      tags: tags
      archives: archive
      sitemap: sitemap
      commonweal: heartbeat
      openstack: cloud
      code&tech: code
      book: book
      essay: file-o

在主题文件夹下有一个language文件夹，下面找到自己设置的语言配置文件，我这里是zh-Hans:

    menu:
      home: 首页
      archives: 归档
      categories: 分类
      tags: 标签
      about: 关于
      search: 搜索
      schedule: 日程表
      sitemap: 站点地图
      commonweal: 公益404
      openstack: 云计算
      code&tech: code-tech
      book: 颜如玉
      essay: 杂谈
在站点配置文件中还要修改：
​    
    # URL
    ## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
    url: http://yuanzizhen.github.io
    root: /
    permalink: :year/:month/:day/:title/
    permalink_defaults:
url要设置为自己的url。

到这里我的第一个博客就全部搭建完成了。

**注意：**

如果要修改了之前发布的文章，最好将对应的public文件夹下的静态网页删除掉，然后再`hexo g`生成，才能产生效果。

如果要设置阅读全文，最好是在文中合适的位置添加`<!-- more -->`

