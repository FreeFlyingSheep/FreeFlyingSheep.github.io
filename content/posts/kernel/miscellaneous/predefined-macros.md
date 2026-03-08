---
title: "预定义宏"
date: 2020-10-10
description: ""
menu:
  sidebar:
    name: "预定义宏"
    identifier: kernel-miscellaneous-predefined-macros
    parent: kernel-miscellaneous
    weight: 100
tags: ["Linux 内核", "C 语言"]
categories: ["Kernel"]
---

根据 GCC CPP 在线文档的 [Predefined Macros](https://gcc.gnu.org/onlinedocs/cpp/Predefined-Macros.html#Predefined-Macros) 章节整理。

<!--more-->

## 简介

GCC 的预处理器 CPP 预定了很多宏，这些宏被称为“预定义宏”（Predefined Macros）。

对于 C 语言，预定义宏包含三类：标准的（standard）、通用的（common）和特定于系统的（system-specific）。

对于 C++ 语言，除了上述三类，还多了一类：命名运算符（named operators）。

## 标准的预定义宏

标准的预定义宏由相关的语言标准指定，因此实现这些标准的所有编译器都可以使用它们（较早的编译器可能无法提供所有的宏），它们的名字都以双下划线开头。

下面列出几个常用的宏：

| 宏 | 含义 |
| --- | --- |
| `__FILE__` | 该宏以 C 字符串常量的形式扩展为当前输入文件的名称（如 `"/usr/local/include/myheader.h"`） |
| `__LINE__` | 该宏以十进制整数常量的形式扩展为当前输入行号。 |
| `__cplusplus` | 当使用 C++ 编译器时，该宏被定义为 `1` |
| `__ASSEMBLER__` | 当预处理汇编语言时，该宏被定义为 `1` |

## 通用的预定义宏

通用的预定义宏是 GNU C 扩展，在 GNU C 和 GNU Fortran 的系统上含义相同，它们的名字都以双下划线开头。

其中，`__GNUC__`、`__GNUC_MINOR__` 和 `__GNUC_PATCHLEVEL__` 被所有使用 C 预处理程序的 GNU 编译器（C，C++，Objective-C 和 Fortran）定义。它们的值分别是编译器的主要版本、次要版本和补丁级别，是整数常量。比如，GCC 版本 `x.y.z` 定义了 `__GNUC__` 为 `x`，`__GNUC_MINOR__` 为 `y`，`__GNUC_PATCHLEVEL__` 为 `z`。

## 特定于系统的预定义宏

C 预处理器通常会预定义一些宏来表示系统和机器类型，显然这些宏在不同目标机器上是不同的。不建议使用特定于系统的预定义宏，最好使用 autoconf 之类的工具检查所需的功能。

## 命名运算符

在 C++ 中，有 11 个关键字，它们是常用标点符号的替代写法。即使在预处理器中，也可以使用这些关键词，如在 `#if` 中拿它们充当运算符。

在 C 中，可以通过包含 `iso646.h` 来要求这些关键字具有 C++ 的含义。该头文件将它们定义为普通的宏，并扩展到相应的标点符号。

下面列出这 11 个命名运算符及对应的标点符号：

| 命名运算符 | 标点符号 |
| :--- | :---: |
| `and` | `&&` |
| `and_eq` | `&=` |
| `bitand` | `&` |
| `bitor` | `|` |
| `compl` | `~` |
| `not` | `!` |
| `not_eq` | `!=` |
| `or` | `||` |
| `or_eq` | `|=` |
| `xor` | `^` |
| `xor_eq` | `^=` |
