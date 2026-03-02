---
title: "Linux 内核常见函数/宏"
date: 2020-10-14
description: ""
menu:
  sidebar:
    name: "Linux 内核常见函数/宏"
    identifier: kernel-introduction-common
    parent: kernel-introduction
    weight: 100
tags: ["Linux 内核", "内核简介"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/zh-cn/posts/kernel/kernel)，内核简介部分，简单介绍 Linux 内核常见的函数及其作用。

<!--more-->

## 内核函数库

显然，在编写内核时，不能使用 C 标准库。内核实现了自己的函数库，位于名为 `lib` 的目录下（对于 `x86` 平台，内核函数库包括 `lib` 和 `arch/x86/lib` 目录）。

内核函数库和 C 标准库中不少函数的功能是一致的。

## 常见函数/宏

### 比较大小

`include/linux/kernel.h`：

```c
/*
 * min()/max()/clamp() macros that also do
 * strict type-checking.. See the
 * "unnecessary" pointer comparison.
 */
#define min(x, y) ({                      \
    typeof(x) _min1 = (x);                \
    typeof(y) _min2 = (y);                \
    (void) (&_min1 == &_min2);            \
    _min1 < _min2 ? _min1 : _min2; })

#define max(x, y) ({                      \
    typeof(x) _max1 = (x);                \
    typeof(y) _max2 = (y);                \
    (void) (&_max1 == &_max2);            \
    _max1 > _max2 ? _max1 : _max2; })

/*
 * ..and if you can't take the strict
 * types, you can specify one yourself.
 *
 * Or not use min/max/clamp at all, of course.
 */
#define min_t(type, x, y) ({              \
    type __min1 = (x);                    \
    type __min2 = (y);                    \
    __min1 < __min2 ? __min1: __min2; })

#define max_t(type, x, y) ({              \
    type __max1 = (x);                    \
    type __max2 = (y);                    \
    __max1 > __max2 ? __max1: __max2; })
```

### 取整

`include/linux/kernel.h`：

```c
/*
 * This looks more complex than it should be. But we need to
 * get the type for the ~ right in round_down (it needs to be
 * as wide as the result!), and we want to evaluate the macro
 * arguments just once each.
 */
#define __round_mask(x, y) ((__typeof__(x))((y)-1))
#define round_up(x, y) ((((x)-1) | __round_mask(x, y))+1)
#define round_down(x, y) ((x) & ~__round_mask(x, y))

#define DIV_ROUND_UP(n,d) (((n) + (d) - 1) / (d))
#define roundup(x, y) ((((x) + ((y) - 1)) / (y)) * (y))
```

### 内存分配

```c
void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *)
```

这两个函数类似于标准库函数 `malloc()` 和 `free()`，其中 `flags` 表示分配标志，在 [Linux 内存管理 (基础部分)](/zh-cn/posts/kernel/memory/model)中介绍。

### 输出信息

```c
int printk(const char *fmt, ...);
```

该函数类似于标准库函数的 `printf()`，但使用时可以在字符串前面加上日志级别，如 `printk(KERN_ERR "bad value %d\n", value);`。如果不指定日志级别，默认是 `KERN_WARNING`，但可能受其他因素影响，最好手动指定。

日志级别定义于 `include/linux/kernel.h`：

```c
#define KERN_EMERG   "<0>" /* system is unusable               */
#define KERN_ALERT   "<1>" /* action must be taken immediately */
#define KERN_CRIT    "<2>" /* critical conditions              */
#define KERN_ERR     "<3>" /* error conditions                 */
#define KERN_WARNING "<4>" /* warning conditions               */
#define KERN_NOTICE  "<5>" /* normal but significant condition */
#define KERN_INFO    "<6>" /* informational                    */
#define KERN_DEBUG   "<7>" /* debug-level messages             */
```

### 缓存对齐

`include/linux/cache.h`：

```c
#ifndef ____cacheline_aligned
#define ____cacheline_aligned __attribute__((__aligned__(SMP_CACHE_BYTES)))
#endif

#ifndef ____cacheline_aligned_in_smp
#ifdef CONFIG_SMP
#define ____cacheline_aligned_in_smp ____cacheline_aligned
#else
#define ____cacheline_aligned_in_smp
#endif /* CONFIG_SMP */
#endif

#ifndef __cacheline_aligned
#define __cacheline_aligned                       \
  __attribute__((__aligned__(SMP_CACHE_BYTES),    \
         __section__(".data.cacheline_aligned")))
#endif /* __cacheline_aligned */

#ifndef __cacheline_aligned_in_smp
#ifdef CONFIG_SMP
#define __cacheline_aligned_in_smp __cacheline_aligned
#else
#define __cacheline_aligned_in_smp
#endif /* CONFIG_SMP */
#endif

/*
 * The maximum alignment needed for some critical structures
 * These could be inter-node cacheline sizes/L3 cacheline
 * size etc.  Define this in asm/cache.h for your arch
 */
#ifndef INTERNODE_CACHE_SHIFT
#define INTERNODE_CACHE_SHIFT L1_CACHE_SHIFT
#endif

#if !defined(____cacheline_internodealigned_in_smp)
#if defined(CONFIG_SMP)
#define ____cacheline_internodealigned_in_smp \
    __attribute__((__aligned__(1 << (INTERNODE_CACHE_SHIFT))))
#else
#define ____cacheline_internodealigned_in_smp
#endif
#endif
```

正如这些宏的字面意思，可以简单理解为按高速缓存行对齐。
