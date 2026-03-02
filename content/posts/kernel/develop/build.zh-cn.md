---
title: "内核编译和调试"
date: 2020-10-19
description: ""
menu:
  sidebar:
    name: "内核编译和调试"
    identifier: kernel-develop-build
    parent: kernel-develop
    weight: 100
tags: ["Linux 内核", "内核开发"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/zh-cn/posts/kernel/kernel)，内核开发部分，简单介绍 Linux 内核编译步骤。

<!--more-->

## 内核源码树

| 目录 | 内容 |
| --- | --- |
| `arch` | 体系结构相关的代码，包括头文件、C 和汇编源文件 |
| `block` | 块设备 I/O 层 |
| `crypto` | 加密层 |
| `Documentation` | 内核源码文档 |
| `drivers` | 设备驱动程序 |
| `firmware` | 使用某些驱动程序而需要的设备固件 |
| `fs` | 文件系统 |
| `include` | 内核头文件 |
| `init` | 内核引导和初始化 |
| `ipc` | 进程间通信 |
| `kernel` | 内核核心组件 |
| `lib` | 内核通用库 |
| `mm` | 内存管理子系统 |
| `net` | 网络子系统 |
| `samples` | 示例，示范代码 |
| `scripts` | 编译内核所用的脚本 |
| `security` | Linux 安全模块 |
| `sound` | 语音子系统，即声卡驱动程序 |
| `tools` | Linux 开发实用工具 |
| `usr` | 早期用户空间的代码 |
| `virt` | 虚拟化基础结构 |

## 配置内核

### Kconfig

内核采用了一种被称为 Kconfig 的配置语言，该配置语言可以解决以下问题：

- 各组件可以持久编译到内核中、可以编译为组件或直接忽略掉（在某些环境下，可能无法将某些组件编译为模块）。
- 在配置选项之间，可能存在相互依赖关系（某些选项只能与一个或多个其他选项连同使用）。
- 必须能够给出一个可用选项列表，供用户从中选择（有些情形，需要提示用户输入编号或类似的值）。
- 必须能够层次化地编排各种配置选项（在一一个树型结构中）。
- 配置选项可能依体系结构而不同。
- 配置语言不应该过于复杂，因为编写配置脚本并非大多数内核程序员喜欢做的事情。

Kconfig 的具体语法和实现不属于本系列学习笔记的范畴，我考虑以后专门开一个专题介绍内核构建系统。

### 配置内核的工具

内核提供了各种不同的工具来简化内核配置。

最简单的是字符界面下的命令行工具：

```bash
make config
```

该工具会逐一遍历所有配置项，要求用户选择 `yes`、`no` 或是 `module`（如果是三选一的话）。由于这个过程往往要耗费掉很长时间，所以，不建议使用该工具配置内核。

常用的工具是基于 ncurse 库编制的图形界面工具：

```bash
make menuconfig
```

或者，使用基于 gtk+ 的图形工具：

```bash
make gconfig
```

### 其他配置方式

除了使用上述三个工具，还可以直接使用内核的默认配置：

```bash
make defconfig
```

不论使用哪种配置方式，最终内核配置文件会位于内核代码树根目录下的 `.config` 文件。

也可以直接复制或修改配置文件 `.config`，这么做之后，应该更新配置文件：

```bash
make oldconfig
```

## 编译内核

### Kbuild

内核使用 GNU make 来编译源代码，这是一个复杂的 Makefile 系统，我们把该系统称为 Kbuild 系统。

同样，Kbuild 的具体实现不属于本系列学习笔记的范畴，我考虑以后专门开一个专题介绍内核构建系统。

### 编译内核的命令

```bash
make [-jn]
```

可选的 `-jn` 参数代表用 `n` 个作业来并行编译。

## 安装内核

### 安装内核映像

把内核映像拷贝到 `/boot` 目录下，修改相应的引导配置。

### 安装内核模块

```bash
make modules_install
```

## 内核编译实例

上面的内容都是在本机上配置编译安装内核的步骤，下面展示一个**交叉编译**内核并安装的实例。

现在我们有一个 x86 体系结构的主机，要为一台龙芯（3A4000）的机器安装内核（假设主机上的“指定目录”为 `../dest`，内核映像将位于 `../dest/boot`，内核模块将位于 `../dest/lib`）：

1. 设置交叉编译工具链的环境变量。
2. 使用默认配置来配置内核：`make ARCH=mips loongson3_defconfig`。
3. 使用 `8` 个作业来并行编译内核：`make ARCH=mips CROSS_COMPILE=mips64el-unknown-linux-gnu- -j8`。
4. 复制内核映像到指定目录：`cp vmlinuz ../dest/boot`。
5. 复制内核配置到指定目录：`cp .config ../dest/boot`（可选）。
6. 安装模块到指定目录：`make ARCH=mips CROSS_COMPILE=mips64el-unknown-linux-gnu- INSTALL_MOD_PATH=../dest -j8 modules_install`。
7. 打包 `../dest` 目录并解包到目标机器。
8. 安装内核映像（假设内核映像文件名为 `vmlinuz-4.x`）：`cp boot/vmlinuz-4.x /boot`（可选地复制内核配置文件到 `/boot`）。
9. 安装内核模块（假设内核模块目录名为 `4.19.150+`）：`cp -r lib/modules/4.19.150+ /lib/modules`。
10. 修改相应的引导配置。

## 内核调试

下面仅列举几个常用的调试方式。

### 通过打印来调试

最好理解，用 `printk()` 函数在相应地方打印信息。

### 借助 GDB 和 KGDB

这部分内容可以参考内核在线文档 [Debugging kernel and modules via gdb](https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html) 和 [Using kgdb, kdb and the kernel debugger internals](https://www.kernel.org/doc/html/latest/dev-tools/kgdb.html)。

### 使用 Git 进行二分搜索

首先，告诉 Git 使用二分搜索：

```bash
git bisect start
```

其次，告诉 Git 出现问题的最早版本（如果当前版本就是，这步可以略过）：

```bash
git bisect bad <commit>
```

然后，告诉 Git 可正常运行的最新版本：

```bash
git bisect good <commit>
```

或者，将上述三步合成一步，直接告诉 Git 起点和终点（注意，终点 `end` 是最近的提交，而起点 `begin` 是更久以前的提交）：

```bash
git bisect start <end> <begin>
```

之后，Git 会自动进行二分搜索，切换到中间的某次提交。如果该提交一切正常，将它标记为“好的”：

```bash
git bisect good
```

反之，将它标记为“坏的”：

```bash
git bisect bad
```

之后，Git 会继续进行二分搜索，直接找到引入问题的那次提交。

最后，退出二分搜索：

```bash
git bisect reset
```

如果确定引发 bug 的源，那么可以指定 Git 仅在与错误相关的目录（`path`）进行二分搜索：

```bash
git bisect start -- <path>
```

这种方法能定位引入 bug 的版本，但如果 bug 已经被修复，希望在上游代码中寻找修复该 bug 的提交，那只能手动二分搜索了。
