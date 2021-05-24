---
title: 如何使用 GitHub Actions 部署 Hexo 博客
tags: 
categories: 杂谈
date: 2021-05-22 21:15:44
---

近期我的电脑坏了，拿出去维修，而我的新电脑上原来的环境都没了，包括 Hexo 博客本地环境。所以就想着要搭建一个可以自动部署的博客环境，省去我来回迁移博客的烦恼。

在网上搜索了一番之后决定使用 GitHub Actions 来部署博客。

<!-- more -->

# GitHub 仓库准备

我们可以把 Hexo 博客分为两类，一类是源文件：

```
.
├── _config.next.yml
├── _config.yml
├── db.json
├── package-lock.json
├── package.json
├── scaffolds
├── source
└── themes
```

一类是通过 `hexo -g` 生成的静态网站文件：

```
public
├── 2020
├── categories
├── tags
....
```

要通过 GitHub 来自动部署博客，我们需要准备两个仓库。

- 私有仓库 blog：用来存放源码文件
- 博客仓库 *.github.io：存放生成的博客站点静态文件

博客仓库这个，相信很多朋友已经有了，所以只需要建立一个私有仓库即可。

## 博客配置

申请完仓库之后，我们需要做一些配置。

1. 首先在本地生成一对密钥文件，例如：

```shell
ssh-keygen -f github-deploy-key
```

然后会在当前目录下看到有两个文件：`github-deploy-key` 和 `github-deploy-key.pub` 文件。

### 配置部署密钥

复制 `github-deploy-key` 文件内容，在 `blog` 仓库 `Settings -> Secrets -> Add a new secret` 页面上添加。

1. 在 `Name` 输入框填写 `HEXO_DEPLOY_PRI`
2. 在 `Value` 输入框填写 `github-deploy-key` 文件内容

复制 `github-deploy-key.pub` 文件内容，在 `your.github.io` 仓库 `Settings -> Deploy keys -> Add deploy key` 页面上添加。

1. 在 `Title` 输入框填写 `HEXO_DEPLOY_PUB`
2. 在 `Key` 输入框填写 `github-deploy-key.pub` 文件内容
3. 勾选 `Allow write access` 选项

# 编写 workflow

这里我们直接使用别人编写好的 workflow 即可。https://github.com/DeppWang/hexo-action

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/20210524234607)

我把示例贴在下面。

另外这个 GitHub 里面的内容最好认真读一下，按照自己的实际情况来修改一下。

```yaml
name: Deploy

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    name: A job to deploy blog.
    steps:
    - name: Checkout
      uses: actions/checkout@v1
      with:
        submodules: true # Checkout private submodules(themes or something else).
    
    # Caching dependencies to speed up workflows. (GitHub will remove any cache entries that have not been accessed in over 7 days.)
    - name: Cache node modules
      uses: actions/cache@v1
      id: cache
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-
    - name: Install Dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      run: npm install
    
    # Deploy hexo blog website.
    - name: Deploy
      id: deploy
      uses: sma11black/hexo-action@v1.0.0
      with:
        deploy_key: ${{ secrets.DEPLOY_KEY }}
        user_name: your github username
        user_email: your github useremail
    # Use the output from the `deploy` step(use for test action)
    - name: Get the output
      run: |
        echo "${{ steps.deploy.outputs.notify }}"

```

这里需要注意要修改几个变量：

- `${{ secrets.DEPLOY_KEY }}`：改成你自己部署的时候起的名字
- user_name: your github username 和 user_email: your github useremail：配置成你自己的名字和地址即可

然后将这些项目 push 到博客即可。

最终提交的可能是这些内容：

```
├── .github
│   └── workflows
├── _config.next.yml
├── _config.yml
├── db.json
├── newblog.sh
├── package-lock.json
├── package.json
├── scaffolds
│   ├── draft.md
│   ├── page.md
│   └── post.md
├── source
│   ├── .DS_Store
│   ├── _drafts
│   ├── _posts
│   ├── images
│   ├── post_bak
│   ├── tags
│   └── uploads
└── themes
    ├── .gitkeep
    └── next
```

这样，我们就将文章 push 到 GitHub 上之后，就可以自动部署了。
