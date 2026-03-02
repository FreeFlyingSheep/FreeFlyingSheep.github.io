---
title: "shell 重定向小技巧"
date: 2020-09-17
description: ""
menu:
  sidebar:
    name: "shell 重定向小技巧"
    identifier: tips-shell-redirect
    parent: tips
    weight: 100
tags: ["shell 脚本"]
categories: ["Tips"]
---

通过重定向让 shell 切换环境后继续执行。

<!--more-->

## 用途

在执行 `su`、`exec` 或者 `chroot` 之类的指令后，继续执行想要的命令，主要是在尝试写自动构建 LFS 的时候遇到的。

## 实例

### `su`

```bash
su - lfs << "EOF"
<command>
EOF
```

### `exec`

```bash
exec /bin/bash -c "<command>"

exec /bin/bash << "EOF"
<command>
EOF

```

### `chroot`

```bash
chroot /mnt/lfs /bin/bash -c "<command>"

chroot /mnt/lfs /bin/bash << "EOF"
<command>
EOF
```
