---
title: "使用 Zsh"
date: 2020-09-23
description: ""
menu:
  sidebar:
    name: "使用 Zsh"
    identifier: tips-zsh
    parent: tips
    weight: 100
tags: ["Zsh", "oh my zsh"]
categories: ["Tips"]
---

使用 Zsh 作为默认 shell，安装 Zsh 和 oh my zsh 框架并完成配置。

<!--more-->

## 安装 Zsh

在 Linux 系统中安装 Zsh，大部分发行版中 Zsh 的包名都为 `zsh`。

## 安装 oh my zsh

参考 [oh my zsh 官网](https://ohmyz.sh/)，使用如下命令安装：

```bash
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

或者：

```bash
sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
```

## 配置 oh my zsh

修改家目录下的 `.zshrc` 文件，将 `ZSH_THEME` 修改为想要的主题，如 `ZSH_THEME="agnoster"`。

某些主题会提示计算机名，如果不希望在当前用户下提示计算机名，可以在 `.zshrc` 文件的末尾添加 `DEFAULT_USER="$(whoami)"`。

## 安装插件和其他主题

安装教程可以参考对应网站上的教程，下面推荐几个插件和主题。

推荐四个插件：

- [额外的命令补全](https://github.com/zsh-users/zsh-completions)
- [代码提示](https://github.com/zsh-users/zsh-autosuggestions)
- [语法高亮](https://github.com/zsh-users/zsh-syntax-highlighting)
- [历史记录的子字符串搜索](https://github.com/zsh-users/zsh-history-substring-search)

推荐一个主题：

- [Powerlevel10k](https://github.com/romkatv/powerlevel10k)
