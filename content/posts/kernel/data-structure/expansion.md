---
title: "GCC 扩展语法"
date: 2020-09-28
description: ""
menu:
  sidebar:
    name: "GCC 扩展语法"
    identifier: kernel-data-structure-expansion
    parent: kernel-data-structure
    weight: 100
tags: ["Linux 内核", "GCC 扩展语法"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，GCC 扩展语法和内核数据结构部分，简单介绍 GCC 扩展语法。

<!--more-->

## `extern "C"`

这是提供给 C++ 编译器的，`extern "C"` 通知编译器使用 C 语言的链接规范，即不需要改变函数名称。由于 C++ 具有函数重载功能，因此 C++ 编译器不能仅使用函数名作为链接的唯一标识，而会在原本的函数名中添加参数相关的信息。

自然，在编译内核时，我们需要使用 C 语言的链接规范。

`include/linux/linkage.h`：

```c
#ifdef __cplusplus
#define CPP_ASMLINKAGE extern "C"
#else
#define CPP_ASMLINKAGE
#endif
```

`__cplusplus` 宏将在使用 C++ 编译器时被定义，这时候需要告知编译器使用 C 语言的链接规范。

## 属性

属性向编译器提供了有关函数或变量用法的详细信息，可能影响代码输出方面的一些细节。这使得编译器可以应用更准确的优化选项，以生成质量更好的代码，或者实现用普通 C 语言无法描述的功能。

属性通过对变量或函数的声明增加前缀或后缀来指定，关键字是 `__attribute__((list))`，下面用几个例子来说明。

### `noreturn`

`include/linux/linkage.h`：

```c
#define ATTRIB_NORET __attribute__((noreturn))
```

`noreturn` 用于指定被调用函数不返回到调用者。该关键词一般用在会触发内核恐慌（panic）的函数，或在结束后通常会关机的函数。使用该属性主要是为了防止编译器在相关代码中发出未初始化变量的警告，或者没有返回值的警告等。

举个简单的例子，首先创建一个文件 `test.c`：

```c
extern void stop(void);

int fun(int n)
{
    if (n < 0)
        stop();
    else
        return 0;
}
```

之后编译该文件（不同版本 GCC 产生的输出可能不同）：

```bash
$ gcc -c -Wall test.c
test.c: In function ‘fun’:
test.c:9:1: warning: control reaches end of non-void function [-Wreturn-type]
    9 | }
      | ^
```

下面我们修改 `test.c`：

```c
extern void stop(void) __attribute__((noreturn));

int fun(int n)
{
    if (n < 0)
        stop();
    else
        return 0;
}
```

再次编译，没有产生任何警告。

### `regparm(number)`

`arch/x86/include/asm/linkage.h`：

```c
#ifdef CONFIG_X86_32
#define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))
```

在 x86-32 平台上，`regparm(number)` 用于指定以寄存器传递参数的个数。`regparm(0)` 即不使用寄存器传递参数。

### `warn_unused_result`

`include/linux/compiler-gcc4.h`：

```c
#define __must_check __attribute__((warn_unused_result))
```

`warn_unused_result` 用于指定函数返回值必须被调用者使用，否则会产生警告。该属性主要用于不检查函数返回值会导致安全问题或者造成 bug 的情况。

## 内联汇编

GCC 允许借助专门的语句，将汇编代码直接集成到 C 代码中，编译器来承担联合代码生成的工作。插入汇编代码是平台相关的，但用于集成汇编语句与 C 代码的机制是平台无关的。

**注意，GCC 无法检查内联汇编的代码是否正确，也不能检查所使用的寄存器是否适用于特定的应用程序，这是程序员的责任。**

插入汇编代码的关键词为 `asm` 或 `__asm__`，语法如下：

```c
asm ("汇编代码（用分号分隔多个语句）"
        : 输出操作数（可选）
        : 输入操作数（可选）
        : 修饰寄存器列表（可选）
);
```

在 x86 平台，汇编代码必须使用 AT&T 格式，这部分内容可以参考《深入理解计算机系统》第二版第三章。

输入和输出操作数通过 `"约束" (变量)` 的形式定义，常用的约束：

| 约束 | 含义 |
| :---: | :--- |
| `r` | 使用一个通用寄存器 |
| `m` | 使用内存中的一个地址 |
| `I` | 定义一个位于 `0-31` 的常数用于 32 位移位操作 |
| `J` | 定义一个位于 `0-63` 的常数用于 64 位移位操作 |

常用的约束修饰符如下表所示：

| 约束修饰符 | 含义 |
| :---: | :--- |
| `=` | 指定操作数是只写的，丢弃前一个值，替换为操作的输出值 |
| `+` | 指定操作数是读写的 |

来看一个简单的例子：

```c
int a = 5;
int b;

asm ("movl %1, %%eax;
      movl %%eax, %0"
      : "=r" (b)
      : "r" (a)
      : "%%eax");
```

该代码段将 `a` 赋值给 `b`，即 `b = a`。其中，`%1` 代表第 `2` 个出现的寄存器（注意从 `0` 开始计数），`%%eax` 代表 `%eax`（`%%` 最终会被转义为一个 `%`），因此 `movl %1, %%eax` 相当于 `%eax = a`。同理，`movl %%eax, %0` 相当于 `b = %eax`。

## `__builtin` 函数

`__builtin` 函数向编译器提供了其他选项，可以执行 C 语言常规能力范围之外的操作，又不必借助内联汇编。

### `__builtin_return_address(0)`

`__builtin_return_address(0)` 返回函数的返回地址，即函数结束时控制流将定位到的目标地址。这是一个特定于体系结构的任务，该函数提供了一个通用的前端。

### `__builtin_expect(long exp, long c)`

`__builtin_expect(long exp, long c)` 函数主要帮助编译器优化分支预测。`exp` 是一个将计算表达式的结果值，`c` 是返回的预期结果。

举个简单的例子，对于如下的 `if` 语句：

```c
if (expression) {
    ...
} else {
    ...
}
```

如果知道在大部分情况下 `expression` 为真，那么这段代码可以优化为：

```c
if (__builtin_expect(expression), 1) {
    ...
} else {
    ...
}
```

这样编译器将通过预先计算第一个分支的方式，来影响处理器的分支预测。

内核中定义了两个宏来优化“很可能”和“不太可能”的分支。

`include/linux/compiler.h`：

```c
#define likely(x) __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)
```

此处了使用双重否定号 (`!!`)，是为了标准化表达式的结果为该函数所预期的 `0` 和 `1`。具体而言，考虑指针和其他表达式的值：

- 空指针 (`NULL`)：第一个否定把 `NULL` 变成了 `1`，第二个否定把 `1` 变成了 `0`。
- 非空指针：第一个否定把非空指针变成了 `0`，第二个否定把 `0` 变成了 `1`。
- 其他表达式的 `0` 值：第一个否定把 `0` 变成了 `1`，第二个否定把 `1` 变成了 `0`。
- 其他表达式的非 `0` 值：第一个否定把非 `0` 值变成了 `0`，第二个否定把 `0` 变成了 `1`。

## 指针运算

通常在 C 语言中，只允许对具有显示类型的指针进行计算，如 `int *`。GNU 编译器拓宽了该限制，支持 `void` 指针和函数指针的运算，加 `1` 的语义是增加 `1` 字节。
