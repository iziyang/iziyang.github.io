---
title: 项目结构
date: 2018-03-28 16:53:06
tags: NodeJS
categories: 编程
---

项目文件都在 `xsc/src/server/` 目录下，目录结构：



```shell
server/
├── dumpVulCSV.js
├── index.js
├── initial.js
├── lib
│   └── smsSender.js
├── model
│   ├── allSitesAreaStat.js
│   ├── allSitesVulStat.js
│   ├── apiKey.js
│   ├── assetServicePort.js
│   ├── assetStatusStat.js
│   ├── baselineResult.js
│   ├── baselineScript.js
│   ├── constants.js
│   ├── flowLog.js
│   ├── flowLogStatis.js
│   ├── index.js
│   ├── monitorPort.js
│   ├── newsCategory.js
│   ├── news.js
│   ├── osStat.js
│   ├── portStat.js
│   ├── questionAnswer.js
│   ├── questionNaire.js
│   ├── servicePort.js
│   ├── serviceStat.js
│   ├── siteAreaStat.js
│   ├── site.js
│   ├── siteVulStat.js
│   ├── tag.js
│   ├── task.js
│   ├── user.js
│   ├── vulDetail.js
│   ├── vul.js
│   └── vulTrash.js
├── plugins
│   ├── auth
│   │   ├── handlers
│   │   │   └── tokenHandler.js
│   │   ├── index.js
│   │   └── routes
│   │       ├── changePassword.js
│   │       ├── hjLogin.js
│   │       ├── index.js
│   │       ├── login.js
│   │       ├── logout.js
│   │       ├── refreshToken.js
│   │       ├── register.js
│   │       └── smsRegister.js
│   ├── redis.js
│   └── worker.js
├── registerPlugins.js
└── routes
    ├── admin
    │   ├── newsCategory.js
    │   ├── news.js
    │   ├── README.md
    │   ├── user.js
    │   └── vulDetail.js
    ├── ass
    │   └── stat
    │       ├── area.js
    │       ├── count.js
    │       ├── index.js
    │       ├── kind.js
    │       ├── level.js
    │       ├── name.js
    │       ├── range.js
    │       ├── top.js
    │       └── vul.js
    ├── baseline
    │   ├── index.js
    │   └── script
    │       ├── add.js
    │       ├── delete.js
    │       ├── detail.js
    │       ├── list.js
    │       └── update.js
    ├── changePassword.js
    ├── common
    │   ├── captcha.js
    │   ├── qiniu_token.js
    │   └── sms.js
    ├── flowLog.js
    ├── flowStatis.js
    ├── index.js
    ├── monitorPort.js
    ├── profile.js
    ├── qanswer
    │   ├── add.js
    │   ├── detail.js
    │   ├── index.js
    │   └── score.js
    ├── qnaire
    │   ├── detail.js
    │   ├── index.js
    │   ├── list.js
    │   └── update.js
    ├── relatedDomain
    │   ├── index.js
    │   ├── list.js
    │   ├── setToSite.js
    │   └── stat.js
    ├── servicePort.js
    ├── site
    │   ├── add.js
    │   ├── areaStat.js
    │   ├── baselineDelete.js
    │   ├── baselineList.js
    │   ├── batchAdd.js
    │   ├── delete.js
    │   ├── detail.js
    │   ├── index.js
    │   ├── list.js
    │   ├── orgList.js
    │   ├── safeStat.js
    │   ├── servicePortList.js
    │   ├── update.js
    │   └── vulList.js
    ├── stat
    │   ├── assetStatus.js
    │   ├── index.js
    │   ├── os.js
    │   ├── port.js
    │   └── service.js
    ├── syncBaselineFailed.js
    ├── syncBaseline.js
    ├── syncFlow.js
    ├── syncVulsFaild.js
    ├── syncVuls.js
    ├── tag
    │   ├── add.js
    │   ├── delete.js
    │   ├── index.js
    │   ├── list.js
    │   └── update.js
    ├── task
    │   ├── addBaseline.js
    │   ├── add.js
    │   ├── deleteTask.js
    │   ├── detail.js
    │   ├── index.js
    │   ├── list.js
    │   ├── retry.js
    │   ├── runningNum.js
    │   ├── status.js
    │   └── vulList.js
    └── vul
        ├── apiKey.js
        ├── downloadDocx.js
        ├── siteVulList.js
        ├── stat
        │   ├── area.js
        │   ├── count.js
        │   ├── index.js
        │   ├── kind.js
        │   ├── level.js
        │   ├── name.js
        │   └── range.js
        ├── vulDetail.js
        └── vul.js
```

`../server/model/` 目录下都是表的定义；

`../plugins/` 目录下是一下登录、认证等的处理；

`../routes/` 目录下包括资产（`/ass`，`/stat`）、基线`baseline`、站点`site`、任务`task`、漏洞`vul`；

所有的文件夹基本都有一个 `index.js` 文件，用来保存所有暴露出来的接口。



