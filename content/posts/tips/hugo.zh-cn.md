---
title: "使用 Hugo 搭建个人博客"
date: 2020-09-23
description: ""
menu:
  sidebar:
    name: "使用 Hugo 搭建个人博客"
    identifier: tips-hugo
    parent: tips
    weight: 100
tags: ["Hugo", "Github Actions", "Github Pages"]
categories: ["Tips"]
---

使用 [Hugo](https://gohugo.io/) 配合 [Github Pages](https://pages.github.com/) 搭建静态博客，利用 [Github Actions](https://github.com/features/actions) 自动部署。

<!--more-->

## 安装 Hugo

### Windows

从 [Hugo Releases](https://github.com/gohugoio/hugo/releases) 页面下载软件压缩包，之后解压到想要的目录，并添加环境变量。

### Linux

在大部分发行版中，直接安装软件包 `hugo` 即可。

## 新建网站

运行以下命令新建一个名为 `Blog` 的网站。

```bash
hugo new site Blog
```

## 使用主题

此处以本人使用的 [LoveIt](https://hugoloveit.com/zh-cn/) 主题为例。

**注意：由于 LoveIt 主题已经不再更新，本博客现在改用 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题。**

使用如下命令初始化 Git 仓库并添加主题子模块：

```bash
git init
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

主题的配置可以参考相应主题的文档。

## 使用 Gitalk

静态博客不具备评论功能，使用 [Gitalk](https://github.com/gitalk/gitalk/) 实现评论功能。

由于 LoveIt 主题支持 Gitalk，只需要注册一个 GitHub Application（位于 Settings -> Developer settings -> OAuth Apps），并在 LoveIt 的配置中设置好相应内容即可启用。

## 关联到 Github

使用 `hugo` 命令会生成静态网站，默认在 `public` 文件夹中。需要 Github 展示的是 `public` 文件夹的内容，而不是整个项目的内容。

可以采用子模块的方式管理这两个项目这样就可以方便地管理这两个项目。

## 设置 Github Pages

在 Github 的 `<username>.github.io` 仓库中设置启用 Github Pages，使用 `master` 分支以及根路径。

保存后等待 Github 完成相应的构建，即可访问 `https://<username>.github.io`，查看个人博客。注意，有时候博客已经完成构建，但该网址仍然无法访问，Github 完成网址的映射可能需要几分钟甚至数个小时。

## 克隆项目

由于项目带子模块，采用如下方式克隆：

```bash
git clone --recurse-submodules <url>
```

或者：

```bash
git clone <url>
cd <project>
git submodule init
git submodule update
```

后两个命令还可以合并为 `git submodule update --init`。

## 自动化脚本

建立脚本来简化常用的操作。

**注意：现在已经可以利用 Github Actions 自动部署了，见 [自动部署](#自动部署)。**

### 一键预览

通常运行 `hugo serve` 即可，但以 LoveIt 主题为例，希望启用一些生成环境的功能，使用如下脚本：

```bash
hugo serve --disableFastRender -e production
```

### 一键部署

同时部署 `Blog` 和 `<username>.github.io` 项目：

```bash
echo "Deploying updates to GitHub..."

cd public
git rm -rf . > /dev/null

cd ..
hugo

cd public
git add .

msg="Rebuild site on `date '+%x %X'`"
if [ -n "$*" ]; then
    msg="$*"
fi
git diff-index --quiet HEAD || git commit -m "$msg"

git push origin master

cd ..
git add .
git diff-index --quiet HEAD || git commit -am "$msg"
git push --recurse-submodules=check origin master
```

每次生成静态网站前强制删了了以前的文件，这是为了应对修改了某些博文所在的目录后，原本的目录会残留的问题，目前没想到更好的解决办法。

### 一键更新

使用 `git pull --rebase --recurse-submodules` 命令更新会让子模块位于分离头指针的状态，分别更新它们：

```bash
echo "Updating public..."
cd public
git pull --rebase

echo "Updating LoveIt..."
cd ../themes/LoveIt
git pull --rebase

echo "Updating Blog..."
cd ../
git pull --rebase
```

## 自动部署

本博客现在已经采用自动部署的方式。

### Github Actions 配置

参考 [GitHub Actions for Hugo](https://github.com/peaceiris/actions-hugo)。

新建 `.github\workflows\gh-pages.yml` 文件：

```yaml
name: Github Pages

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true    # Use Hugo extended

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          force_orphan: true
```

现在 Github Actions 会在每次 `push` 后自动编译网站，并将生成的 `public` 文件夹中的内容上传到 `gh-pages` 分支。

### Github Pages 配置

启用 Github Pages，使用 `gh-pages` 分支以及根路径。

### 依赖自动更新

新建 `.github\dependabot.yml` 文件：

```yaml
version: 2
updates:
  # Enable version updates for github-actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "daily"
    labels:
      - "dependabot"
    commit-message:
      prefix: "github-actions"
```

现在 [Github Dependabot](https://dependabot.com/) 会每日自动检测 `github-actions` 脚本更新。

### 自动部署到另一个仓库

**再次更新：现在我是这么干的！**

把博客源码和发布的内容放一个仓库总显得有些难受，大部分情况下我们希望博客源码仓库是私有的。

`.github\workflows\gh-pages.yml` 文件修改如下：

```yaml
name: Github Pages

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true    # Use Hugo extended

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: FreeFlyingSheep/FreeFlyingSheep.github.io
          publish_branch: gh-pages
          publish_dir: ./public
          force_orphan: true
```

生成部署用的 ssh key：

```bash
ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
```

在博客源码仓库的 `Settings-> Secrets (Actions secrets)` 里添加 `gh-pages` 文件的内容，且名称为 `ACTIONS_DEPLOY_KEY`。
在要部署的仓库的 `Settings->Deploy keys` 里添加 `gh-pages.pub` 文件的内容，勾选 `Allow write access`，名称随意。

之后每次 `git push` 到博客源码仓库，Github Actions 就会自动推送到发布仓库。
