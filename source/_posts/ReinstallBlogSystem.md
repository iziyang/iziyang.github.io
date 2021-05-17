---
title: 重新部署博客系统
date: 2017-12-12 22:52:52
tags: [hexo,blog]
categories: 编程
---

之前不知道什么原因，导致我的 hexo 博客不能用了，然后就重新部署了一下，参照我之前的配置：



https://yuanzizhen.github.io/2017/03/11/%E7%94%A8Hexo%E5%9C%A8Github%E4%B8%8A%E9%9D%A2%E9%83%A8%E7%BD%B2%E5%8D%9A%E5%AE%A2/

但是依然遇到了很多坑，特此写在这里。

1、使用 npm 安装 hexo 的时候屡次不成功。解决办法：

使用淘宝 NPM 镜像：

```
npm install -g cnpm --registry=https://registry.npm.taobao.org
cnpm install -g hexo
```

参考：http://www.runoob.com/nodejs/nodejs-npm.html

2、github 生成 ssh 失败

```
$ ssh-keygen -t rsa -C C "iyuanzz@qq.com"
Too many arguments.
usage: ssh-keygen [-q] [-b bits] [-t dsa | ecdsa | ed25519 | rsa]
                  [-N new_passphrase] [-C comment] [-f output_keyfile]
       ssh-keygen -p [-P old_passphrase] [-N new_passphrase] [-f keyfile]
       ssh-keygen -i [-m key_format] [-f input_keyfile]
       ssh-keygen -e [-m key_format] [-f input_keyfile]
       ssh-keygen -y [-f input_keyfile]
       ssh-keygen -c [-P passphrase] [-C comment] [-f keyfile]
       ssh-keygen -l [-v] [-E fingerprint_hash] [-f input_keyfile]
       ssh-keygen -B [-f input_keyfile]
       ssh-keygen -D pkcs11
       ssh-keygen -F hostname [-f known_hosts_file] [-l]
       ssh-keygen -H [-f known_hosts_file]
       ssh-keygen -R hostname [-f known_hosts_file]
       ssh-keygen -r hostname [-f input_keyfile] [-g]
       ssh-keygen -G output_file [-v] [-b bits] [-M memory] [-S start_point]
       ssh-keygen -T output_file -f input_file [-v] [-a rounds] [-J num_lines]
                  [-j start_line] [-K checkpt] [-W generator]
       ssh-keygen -s ca_key -I certificate_identity [-h] [-U]
                  [-D pkcs11_provider] [-n principals] [-O option]
                  [-V validity_interval] [-z serial_number] file ...
       ssh-keygen -L [-f input_keyfile]
       ssh-keygen -A
       ssh-keygen -k -f krl_file [-u] [-s ca_public] [-z version_number]
                  file ...
       ssh-keygen -Q -f krl_file file ...
```

解决办法：

删除一个 `C` 解决

2、使用命令`hexo d` 无法部署

![](http://ww1.sinaimg.cn/large/c552abe7ly1fmedol9rrtj207m02f3yg.jpg)

解决办法

重新设置：

```
npm install hexo-deployer-git --save
```

最终部署成功！