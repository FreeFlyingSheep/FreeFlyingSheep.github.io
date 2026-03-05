---
title: "内存管理补充"
date: 2020-07-27
description: ""
menu:
  sidebar:
    name: "内存管理补充"
    identifier: kernel-memory-supplement
    parent: kernel-memory
    weight: 100
tags: ["Linux 内核", "内存管理"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，内存管理部分，内存管理的一些补充说明。

<!--more-->

## 分配函数的选择

如果需要连续的物理页，可以使用某个低级页分配器（[分区页框分配器](/posts/kernel/memory/continuous#分区页框分配器)里请求页框和释放页框的函数/宏）或 `kmalloc()` 函数。

对于中断处理程序和其他不能睡眠的代码段，使用 `GFP_ATOMIC` 表示不进行睡眠的高优先级分配。

对于可以睡眠的代码，使用 `GFP_KERNEL` 表示如果有必要可以进行睡眠。

如果想从高端内存进行分配，可以使用 `alloc_pages()` 函数。
该函数返回一个指向 `struct page` 结构的指针，而不是一个指向某个逻辑地址的指针。因为高端内存很可能没有被映射，要获得真正的指针，需要调用 `kmap()` 函数把高端内存映射到内核的逻辑地址空间。

如果不需要物理上连续的页，而仅仅需要虚拟地址上连续的页，可以使用 `vmalloc()` 函数（但该函数相对 `kmalloc()` 有一定的性能损失）。

如果要创建和撤销很多大的数据结构，可以考虑建立 slab 高速缓存。

## `vmalloc()` 和 `kmalloc()`

`vmalloc()` 为了把物理上不连续的页转换为虚拟地址空间上连续的页，必须专门建立页表项，而且因为它们物理上是不连续的，所以必须一个一个进行映射，这会导致比直接内存映射大得多的 TLB 抖动。

尽管在某些情况下才需要物理上的连续内存块，但出于性能的考虑，很多内核代码都使用 `kmalloc()`。除非在不得已时才使用 `vmalloc()`，比如动态装载模块时，需要获得大块内存。

## slub 分配器

slub 分配器逐渐取代了 slab 分配器，成为默认的内存分配器。

slub 分配器在 2.6.22 版本引入，它具有设计简单、代码精简、额外内存占用率小、扩展性高，性能优秀、方便调试等很多优点。

## 内存管理相关函数调用关系

分配连续内存的函数调用关系大致如下：

![分配连续内存的函数调用关系](/images/kernel/memory/alloc_memory.png)

释放连续内存的函数调用关系大致如下：

![释放连续内存的函数调用关系](/images/kernel/memory/free_memory.png)
