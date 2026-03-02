---
title: "宏的陷阱和使用技巧"
date: 2020-09-10
description: ""
menu:
  sidebar:
    name: "宏的陷阱和使用技巧"
    identifier: kernel-miscellaneous-macros-pitfalls
    parent: kernel-miscellaneous
    weight: 100
tags: ["Linux 内核", "C 语言"]
categories: ["Kernel"]
---

根据 GCC CPP 在线文档的 [Macro Pitfalls](https://gcc.gnu.org/onlinedocs/cpp/Macro-Pitfalls.html#Macro-Pitfalls) 章节整理。

<!--more-->

## `do ... while(0)`

### `do ... while(0)` 的实例

在内核中经常包括以下种类的宏：

```c
#define ELF_PLAT_INIT(_r, load_addr)    \
    do {                                \
    _r->bx = 0; _r->cx = 0; _r->dx = 0; \
    _r->si = 0; _r->di = 0; _r->bp = 0; \
    _r->ax = 0;                         \
} while (0)
```

### `do ... while(0)` 的作用

如果把上面的宏用于 `if` 语句或类似的语言要素，比如：

```c
if (...)
    ELF_PLAT_INIT(...);
```

此时如果不使用 `do ... while(0)` 来封装，就会出现问题，只有宏的第一行是归入 `if` 语句体的，这显然不是我们期望的结果。

使用 `do ... while(0)` 封装，就可以避免这种问题，保证宏的所有内容都被归于 `if` 语句体，而多余的循环语句也会被编译器自动优化掉。

补充：那么，能不能直接写成如下形式呢：

```c
#define ELF_PLAT_INIT(_r, load_addr)    \
{                                       \
    _r->bx = 0; _r->cx = 0; _r->dx = 0; \
    _r->si = 0; _r->di = 0; _r->bp = 0; \
    _r->ax = 0;                         \
}
```

乍一看似乎没问题，考虑下面的用法：

```c
if (...)
    ELF_PLAT_INIT(...);
else
    ...
```

这将被扩展成如下形式：

```c
if (...) {
    _r->bx = 0; _r->cx = 0; _r->dx = 0;
    _r->si = 0; _r->di = 0; _r->bp = 0;
    _r->ax = 0;
};
else
    ...
```

我们注意到大括号后面多了个分号，导致 `else` 悬空了，所以省掉 `do ... while(0)` 是不行的。

## `({ … })`

### `({ … })` 的实例

```c
#define min(X, Y)                \
({ typeof (X) x_ = (X);          \
   typeof (Y) y_ = (Y);          \
   (x_ < y_) ? x_ : y_; })
```

### `({ … })` 的作用

按照常规写法，我们会如此定义求最小值的宏：

```c
#define min(X, Y)  ((X) < (Y) ? (X) : (Y))
```

但当我们把具有副作用的表达式作为参数时，如 `next = min (x + y, foo (z));`，该代码会被扩展为 `next = ((x + y) < (foo (z)) ? (x + y) : (foo (z)));`，表达式 `x + y` 或 `foo(z)` 将被执行两遍，这显然不是我们希望的。

标准 C 中没有办法能解决这个问题（硬要说的话，要么小心地使用这些宏，要么采用内联函数）。

GCC 扩展语法提供了 `({ … })` 来生成一个用作表达式的复合语句，其值是最后一个语句的值。这允许我们在其中定义局部变量，把参数赋值给局部变量来保证它们只被计算一次，正如实例中的 `min` 宏。注意，局部变量的名称后面带有下划线，是为了以减少与其他变量发生冲突的可能性，但这是无法避免的。

## 自引用的宏（Self-Referential Macros）

### 自引用的宏简介

一个自引用的宏是名字出现在其定义中的宏。

为了避免无限的展开，自引用的宏只会被扩展一次，举个例子：

```c
#define foo (4 + foo)
```

在代码中用到的 `foo` 只会被扩展为 `(4 + foo)`，而不会再被扩展为 `(4 + (4 + foo))`。

### 自引用的宏的实例

在代码中，经常看到在枚举类型中穿插自引用的宏定义，比如 `/usr/include/dirent.h` 头文件：

```c
/* File types for `d_type'.  */
enum
{
    DT_UNKNOWN = 0,
# define DT_UNKNOWN DT_UNKNOWN
    DT_FIFO = 1,
# define DT_FIFO    DT_FIFO
    DT_CHR = 2,
# define DT_CHR     DT_CHR
    DT_DIR = 4,
# define DT_DIR     DT_DIR
    DT_BLK = 6,
# define DT_BLK     DT_BLK
    DT_REG = 8,
# define DT_REG     DT_REG
    DT_LNK = 10,
# define DT_LNK     DT_LNK
    DT_SOCK = 12,
# define DT_SOCK    DT_SOCK
    DT_WHT = 14
# define DT_WHT     DT_WHT
};
```

### 自引用的宏的作用

根据官方文档，此处自引用的作用是对于用 `enum` 定义的数字常量，也可以使用 `#ifdef` 进行预处理。

根据 Stack Overflow 上[相关提问](https://stackoverflow.com/questions/8588649/what-is-the-purpose-of-a-these-define-within-an-enum)的回答，这可能是一个历史遗漏问题，原本的代码采用 `#define` 定义常量，而后续改用 `enum` 定义，为了与先前的代码兼容，额外使用自引用的宏，我倾向于这种观点。

## 参数预扫描

### 参数预扫描介绍

除非将宏的参数是一个字符串化的（如 `#x`）或被连接到其他标记上（如 `x##y`），否则它们在被替换为宏主体之前必须经过完整的扩展。替换后，预处理器将再次扫描整个宏主体（包括替换的参数），以进一步扩展。

特别地，对于自引用的宏，若它在第一次扫描中不被扩展，那么在第二次扫描中它也不会被扩展。

可以简单地把参数预扫描理解为先扩展宏的参数（第一次扫描），再扩展宏本身（第二次扫描），如果宏的主体包含 `#` 或 `##`，则不扩展参数。

### 参数预扫描的实例

Linux 2.6.34 内核中的 `include/linux/compiler-gcc.h` 文件中包含如下的代码，可以根据 GCC 版本导入相应的头文件：

```c
#define __gcc_header(x) #x
#define _gcc_header(x) __gcc_header(linux/compiler-gcc##x.h)
#define gcc_header(x) _gcc_header(x)
#include gcc_header(__GNUC__)
```

若使用 GCC 4.x 版本编译这段代码，最终会生成 `#include "linux/compiler-gcc4.h"`（`4` 由 `__GNUC__` 提供，这是 GCC 的预定义宏，代表 GCC 主版本号），以下是扩展的步骤：

1. 第一次扫描 `gcc_header(__GNUC__)`，扩展为 `gcc_header(4)`。
2. 第二次扫描 `gcc_header(4)`，扩展为 `_gcc_header(4)`。
3. 第一次扫描 `_gcc_header(4)`，发现 `##`，不扩展。
4. 第二次扫描 `_gcc_header(4)`，扩展为 `__gcc_header(linux/compiler-gcc4.h)`。
5. 第一次扫描 `__gcc_header(linux/compiler-gcc4.h)`，发现 `#`，不扩展
6. 第二次扫描`__gcc_header(linux/compiler-gcc4.h)`，扩展为 `"linux/compiler-gcc4.h"`。

注意，`_gcc_header` 宏是必要的，考虑下面的写法：

```c
#define __gcc_header(x) #x
#define gcc_header(x) __gcc_header(linux/compiler-gcc##x.h)
#include gcc_header(__GNUC__)
```

这段代码最终会生成 `#include "linux/compiler-gcc__GNUC__.h"`，以下是扩展的步骤：

1. 第一次扫描 `gcc_header(__GNUC__)`，发现 `##`，不扩展。
2. 第二次扫描 `gcc_header(__GNUC__)`，扩展为 `__gcc_header(linux/compiler-gcc__GNUC__.h)`。
3. 第一次扫描 `__gcc_header(linux/compiler-gcc__GNUC__.h)`，发现 `#`，不扩展。
4. 第二次扫描 `__gcc_header(linux/compiler-gcc__GNUC__.h)`，扩展为 `"linux/compiler-gcc__GNUC__.h"`。
