---
title: "内核开发相关知识"
date: 2020-10-22
description: ""
menu:
  sidebar:
    name: "内核开发相关知识"
    identifier: kernel-develop-develop
    parent: kernel-develop
    weight: 100
tags: ["Linux 内核", "内核开发"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，内核开发部分，简单介绍 Linux 内核的开发结构的相关知识，以提供参考资料为主。

<!--more-->

## 内核开发

内核文档中提供了大量关于内核开发的内容，具体可以参考 [The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/)。

## 开发结构

### 命令链

内核所有活动组件都有一个维护者，维护者在 `MAINTAINERS` 文件中给出，具体内容可以直接看该文件头部的说明文字。

### 开发周期

在一个新的内核版本发布后，Linus Torvalds 会打开一个合并窗口（merge window），在比较短的一段时间内保持开放，大约两星期。新代码通常只能在这段时间内加入。

合并窗口关闭时，候选的发布版内核也准备好了。候选发布版提供了一个机会，可以测试各项修改之间的交互，以及识别并修复bug。

在一切都稳定以后，一个新的内核版本就发布了。

### 在线资源

- <https://www.kernel.org/>：包含了内核源代码及许多基本的用户空间工具。
- <https://git.kernel.org/>：Git 源代码的存储库。
- <https://lwn.net/>：内核开发过程方面的首要信息源，收集了 Linux 开发所有方面的有趣新闻以及 IT 社区中的相关事件，而优秀的研究文章对各个项目的发展现状给出了深刻的见解。

## 编码风格

具体可以参考 [Linux kernel coding style](https://www.kernel.org/doc/html/latest/process/coding-style.html)。

## 补丁结构

现在建议使用 Git 来管理补丁。

提交补丁可以参考 [Submitting patches: the essential guide to getting your code into the kernel](https://www.kernel.org/doc/html/latest/process/submitting-patches.html)。
