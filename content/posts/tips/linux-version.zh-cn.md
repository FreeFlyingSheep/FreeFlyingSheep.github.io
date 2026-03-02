---
title: "查看 Linux 版本"
date: 2020-09-21
description: ""
menu:
  sidebar:
    name: "查看 Linux 版本"
    identifier: tips-linux-version
    parent: tips
    weight: 100
tags: ["Linux"]
categories: ["Tips"]
---

查看 Linux 发行版和内核版本的多种方式。

<!--more-->

## Linux 发行版

以下指令均能查看 Linux 发行版信息：

```bash
uname -a
cat /etc/os-release
cat /etc/lsb-release
```

其中，`/etc/lsb-release` 仅推荐在 `/etc/os-release` 不存在的时候访问。

## Linux 内核

以下指令均能查看 Linux 内核信息：

```bash
uname -a
cat /proc/version
```

其中，`#` 后面的数字代表编译次数。如果是手动编译的内核，它应该与源码目录下的 `.version` 文件保持一致。同时，顶层 `Makefile` 的前几行代表内核版本信息。
