---
title: "MIPS（补充内容）"
date: 2020-07-15
description: ""
menu:
  sidebar:
    name: "MIPS（补充内容）"
    identifier: mips-supplement
    parent: mips
    weight: 100
tags: ["MIPS", "汇编", "体系结构"]
categories: ["MIPS"]
---

针对 [MIPS 汇编](/posts/mips/assembly") 和 [MIPS 体系结构](/posts/mips/see-mips-run) 的补充内容。

<!--more-->

## ABI

以下内容来自 *[MIPS Assembly Language Programmer’s Guide](https://courses.cs.washington.edu/courses/cse410/05sp/misc/MIPS-ASM-007-2418-006.pdf)* 。

对于 n64，部分整数寄存器的约定见下表：

| 寄存器名称 | 助记符 | 用途 | 保存者 |
| --- | --- | --- | --- |
| `$0`  | `zero` | 永远返回0 |  |
| `$1` | `at` | 汇编器保留 | 调用者 |
| `$2`-`$3` | `v0`-`v1` | 函数返回值 | 调用者 |
| `$4`-`$11` | `a0`-`a7` | 子程序参数 | 调用者 |
| `$12`-`$15` | `t4`-`t7` | 临时寄存器 | 调用者 |
| `$16`-`$23` | `s0`-`s7` | 保存寄存器 | 被调用者 |
| `$24` | `t8` | 临时寄存器 | 调用者 |
| `$25` | `t9` | 临时寄存器 | 调用者 |
| `$26`-`$27` | `k0`-`k1` | 内核保留 |
| `$28` | `gp` | 全局指针 | 被调用者 |
| `$29` | `sp` | 栈指针 | 被调用者 |
| `$30` | `s8` | 帧指针 | 被调用者 |
| `$31` | `ra` | 返回地址 | 调用者 |

对于整数寄存器，o32 和 n64 的部分区别见下表：

| 属性 | o32 | n64 |
| --- | --- | --- |
| 寄存器长度 | 32 位 | 64 位 |
| 当参数为结构体时，通过寄存器传递的大小 | 32 位 | 64 位 |
| 参数寄存器保存者 | 调用者 | 被调用者（只在需要时保存） |
| 当返回值为结构体时，通过寄存器返回的大小 | 不保存 | 最多 128 位 |
| 当返回值为结构体时，第一个参数由 `$2` 返回 | 是 | 否 |
| 返回地址存放位置 | `$31`，支持 `.mask` 指令 | 任意寄存器 |
| 全局指针保存者 | 调用者保存 | 被调用者保存 |

## 虚拟地址空间

以下内容来自 *[The MIPS64 and microMIPS64 Privileged Resource Architecture v6.03](https://s3-eu-west-1.amazonaws.com/downloads-mips/documents/MD00091-2B-MIPS64PRA-AFP-06.03.pdf)* 。

虚拟地址空间如下图所示：

![虚拟地址空间](/images/mips/virtual_address_space.png)

特别地，对于 `xkphys` 空间，虚拟地址的格式如下：

![`xkphys` 空间下虚拟地址的格式](/images/mips/address_interpretation.png)

针对 `CCA` 部分，具体描述如下：

![`CCA` 的具体描述](/images/mips/cacheabillity_and_coherency.png)

根据上表描述的属性，`xkphys` 空间包括八个地址范围，每个地址范围提供了进入物理内存的 $2^{PABITS}$ 字节的窗口，因此不会使用TLB转换地址。

## Linux/MIPS 的内存管理

- 32 位的 Linux/MIPS 内核假定整个低内存可通过 `kseg0` 访问，这一空间但最多 512MB。通常会保留该地址空间的一部分供其他使用，因此把低端内存限制为256MB，超出此范围的内存通过高端内存来访问。
- 64位的 Linux/MIPS 内核通过xkphys访问低端内存。由于 xkphys 的大小足够，整个物理内存是可以直接访问的，因此不需要高端内存。

## 参考文档

MIPS 体系结构的文档可在官网上找到：

- [MIPS32 Architecture](https://www.mips.com/products/architectures/mips32-2/)
- [MIPS64 Architecture](https://www.mips.com/products/architectures/mips64/)
