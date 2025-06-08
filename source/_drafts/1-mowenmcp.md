---
title: 如何在 Trae 中用 Markdown 写墨问
tags: 墨问
categories: 杂谈
date: 2025-06-08 09:29:30
description: 如何在 Trae 中用 Markdown 写墨问。
---

在墨问更新了云端 MCP 之后，就解决了一个问题，那就是如何把 Markdown 内容导入到墨问中。在之前，可能需要手动复制到墨问，然后再一行行调整格式、间距等等。而现在，只需要用一行指令，就能让大模型自动调用墨问 MCP 来帮你生成一篇墨问笔记。

我之前写 Markdown 是用 typora 这个工具的，但是它不具备 AI 能力，所以这篇文章，就写写如何在 Trae 中用 Markdown 写墨问，**让它既具备 typora 的能力，又能调用大模型的能力。**

typora 是一个很好用的工具，**它具备所见即所得的能力，同时，我还依赖它的自定义图床**。我有一个自定义图床，在 typora 只需要调用我的脚本就能实现粘贴图片然后自动上传的功能。

![](https://s3plus.meituan.net/v1/mss_f32142e8d47149129e9550e929704625/yzz-test-image/90d1d89f95fc4b8a8573e3ba12452945)  

现在，我需要在 Trae 中进行配置，让它能够具备这两个能力，我们可以使用两个插件完成。

一个是 Markdown All in One，在 Trae 的插件市场中可以找到，它能实现预览的能力。

第二个是 Markdown Image，这个在 Trae 的插件市场中是找不到的，因为 Trae 使用 open-vsx 的镜像作为市场，但是没关系，Trae 给出了解决方案，对于找不到的插件，我们可以从 vscode 的镜像市场中下载，然后拖入到插件市场中即可，我把链接附在下面。

https://marketplace.visualstudio.com/_apis/public/gallery/publishers/hancel/vsextensions/markdown-image/1.1.49/vspackage

然后，在 Trae 的 AI 对话框中，告诉它把这篇文章存为墨问笔记，就成功发表啦，无比丝滑。