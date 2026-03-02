---
title: "Git 打补丁"
date: 2020-08-11
description: ""
menu:
  sidebar:
    name: "Git 打补丁"
    identifier: git-patch
    parent: git
    weight: 100
tags: ["Git"]
categories: ["Git"]
---

整理使用 Git 打补丁的方法，补充一些常用指令，并给出一个 Git 命令的综合应用实例。

<!--more-->

## 导出补丁

### `git diff`

```bash
git diff <branch1>[:<file1>] <branch2>[:<file2>] > <patch>
```

使用 `git diff` 导出的补丁，不包含提交（commit）信息。

### `git format-patch`

导出单个补丁：

```bash
git format-patch -1 <commit> [-o <dir>]
```

导出多个连续的补丁：

```bash
git format-patch <commit1>[..<commit2>] [-o <dir>]
```

**注意，导出的区间是 `(commit1, commit2]`。**

使用 `git format-patch` 导出的补丁，默认包含提交信息。

## 应用补丁

### `git apply`

直接应用补丁：

```bash
git apply <patch>
```

不应用补丁，而是检查补丁能否被直接应用：

```bash
git apply --check <patch>
```

强制应用补丁，遇到冲突会生成 `.rej` 文件：

```bash
git apply --reject <patch>
```

### `git am`

```bash
git am <patch> ...
```

在遇到冲突时，可以使用如下方法处理：

- `git am --skip`：跳过当前补丁。
- `git am --abort`：终止打补丁并恢复打补丁前的状态。**注意，如果应用了多个补丁，会直接恢复到最初状态，而不是仅终止当前的补丁**。
- `git am --continue`：手动解决冲突后，继续下一个补丁。

## 其他指令

### 清理未跟踪的文件

可以先执行以下命令查看要被清理的文件：

```bash
git clean -n
```

然后执行以下命令清理未跟踪的文件：

```bash
git clean -f
```

### 撤销提交

有时我们需要取消某次具体的提交，这次提交与当前头指针可能已经相隔了多次提交，因此我们不方便直接用 `git reset` 回滚，此时可以使用如下命令撤销单次提交：

```bash
git revert <commit>
```

也可以使用如下命令撤销一系列连续的提交：

```bash
git revert <commit1>..<commit2>
```

注意撤销范围是 `(<commit1>, <commit2>]`。

## Git 命令的综合应用实例

### 简介

现在有一个练手任务：在龙芯上移植 3.16.85 版本的内核到 3.17.0 版本。

有一个修改过的 linux 内核，其中 `linux-3.16` 分支由于已经加入了补丁，所以能在龙芯机器上运行，现在需要把该分支从 3.16.85 版本开始的补丁，应用到 3.17.0 版本上，最终让 3.17.0 版本也能在龙芯机器上运行。

### 获取项目

首先，从指定地址克隆项目（修改过的 linux 内核）。

```bash
git clone ...
```

### 查看分支

使用 `git branch` 查看分支：

```bash
$ git branch
  linux-3.16
* master
```

发现默认分支是 `master`。

### 切换分支

需要先切换到 `linux-3.16` 分支：

```bash
git checkout linux-3.16
```

### 查看提交信息

查看提交信息，确定从哪个提交开始导出补丁：

```bash
$ git log --oneline
...
a804eddbca6c MIPS: Fix restart of indirect syscalls
8ac8e9653550 Sync code to Linux-3.16.85 from upstream
19583ca584d6 (tag: v3.16) Linux 3.16
```

3.16.85 版本对应的提交的哈希值是 `8ac8e9653550`。

### 导出相应补丁

把从 `8ac8e9653550` 开始（不包括 `8ac8e9653550`）的提交以补丁形式导出到上级目录的 `patch` 文件夹。

```bash
git format-patch 8ac8e965355 -o ../patch
```

### 查看标签

之前使用 `git branch` 发现项目并不包含 `linux-3.17` 分支，使用 `git tag` 查看标签：

```bash
$ git tag
...
v3.16
v3.16-rc1
v3.16-rc2
v3.16-rc3
v3.16-rc4
v3.16-rc5
v3.16-rc6
v3.16-rc7
v3.17
v3.17-rc1
...
```

`v3.17` 标签就对应需要的 3.17.0 版本。

### 创建并切换分支

从 `v3.17` 标签创建新分支 `linux-3.17`，并切换到该分支：

```bash
git checkout -b linux-3.17 v3.17
```

### 应用相应补丁

```bash
git am ../patch/*
```

#### 跳过不需要的补丁

```bash
git am --skip
```

#### 手动解决冲突

对于小补丁，直接看补丁文件，修改相应内容即可。

对于大补丁，我采用如下思路解决冲突：

1. 使用 `git apply --reject <patch>` 强制打上补丁。
2. 根据 `.rej` 文件修改相应内容。
3. 删除 `.rej` 文件。
4. 使用 `git am --continue` 继续下一个补丁。

### 其他补丁

在移植过程中，发现有些编译问题已经被其他人解决，可以直接导入相关补丁。

此时使用 `git diff` 导出补丁，然后用 `git apply` 应用，手动解决冲突。
