---
title: "内存寻址"
date: 2020-10-29
description: ""
menu:
  sidebar:
    name: "内存寻址"
    identifier: kernel-memory-address
    parent: kernel-memory
    weight: 100
tags: ["Linux 内核", "内存管理", "内存寻址"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，内存管理部分，简单介绍内存寻址。

<!--more-->

## 简介

这部分内容和体系结构密切相关，而且涉及了大量过时的内容，我对这一块内容的兴趣也不大，所以这里只对 Intel x86 体系结构做很简单的介绍。

更多资料建议参考 [Intel 官方文档](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)和 [AMD 官方文档](https://developer.amd.com/resources/developer-guides-manuals/)。

## 分段内存模式

以下内容仅针对 Intel x86_32 体系结构。

### 逻辑地址

逻辑地址（logical address）由一个段（segment）和偏移量（offset 或 displacement）组成。

### 线性地址（虚拟地址）

线性地址（linear address）也被称作虚拟地址（virtual address），是 `32` 位的无符号整数。

### 物理地址

用于内存芯片级内存单元寻址。

### 地址间的关系

```text
逻辑地址 <--- 分段单元 ---> 线性地址（虚拟地址）<--- 分页单元 ---> 物理地址
```

再次强调，**线性地址就是虚拟地址**，这两者是完全等价的。

## 平坦内存模式

在 x86_64 体系结构中，分段被禁用了，内存是平坦的。

线性地址是 `48` 位的。

尽管 x86_64 理论上可以支持 `64` 位的线性地址，但目前 `48` 位已经足够使用了，所以处理器硬件通常也只提供了 `48` 条地址线。

## 分段

现在是 64 位处理器的时代，分段基本被废弃了，这里只给出几个参考资料：

- [GDT](https://wiki.osdev.org/GDT)。
- [LDT](https://wiki.osdev.org/LDT)。
- [IDT](https://wiki.osdev.org/IDT)。
- [TSS](https://wiki.osdev.org/TSS)。

相关的代码我准备放到系统启动部分一起介绍，见 [TODO](/posts/kernel/todo)。

## 分页

对于 32 位系统，两级页表已经足够了；而对于 64 位系统，往往需要更多的分页级别。

Linux 采用了**四级分页模型**，如下图所示：

![Linux 分页模式](/images/kernel/memory/paging.png)

相关的宏和函数基本位于 `arch/x86/include/asm/pgtable.h`，大部分看名字就能猜测出功能，实现也比较简单，这里不准备展开了。
