---
title: "移植 RT-Thread 到 LoongArch64"
date: 2023-02-18
description: ""
menu:
  sidebar:
    name: "移植 RT-Thread 到 LoongArch64"
    identifier: rt-thread-loongarch64
    parent: rt-thread
    weight: 100
tags: ["RT-Thread", "LoongArch"]
categories: ["RT-Thread"]
---

简单介绍把 RT-Thread 移植到 LoongArch64 的过程。

<!--more-->

## 前言

[移植 RT-Thread 到 LoongArch32](/posts/rt-thread/loongarch32) 中已经介绍了移植 RT 的思路和方式，不再重复介绍。

本文主要负责填坑，提供编译交叉工具链、newlib 库相关的移植过程，相关代码已经上传 Github。

## Binutils

Binutils 已经提供了 `loongarch64-unknown-elf` 的支持，直接下载编译即可。

这里我使用了主线的 Binutils，commit 号是 `e8eca7a6b602290bb3f50728432d524577ade727`。

```bash
git clone https://github.com/FreeFlyingSheep/binutils-gdb.git
cd binutils-gdb
mkdir build
cd build
../configure --target=loongarch64-unknown-elf
make
make install
```

## GCC（Bootstrap）

GCC 暂时没添加 `loongarch64-unknown-elf` 的支持，需要稍作修改。

主要是在 `gcc/config.gcc` 和 `libgcc/config.host` 中添加 `loongarch*-*-elf*)` 的识别，然后添加 `gcc/config/loongarch/elf.h`、`libgcc/config/loongarch/crti.S` 和 `libgcc/config/loongarch/crtn.S` 三个文件。

我这里基于了主线修改，主线的 commit 号是 `a7d8c40484c31a74a2c2bb17d835d60ba7dd8d29`。

```bash
git clone https://github.com/FreeFlyingSheep/gcc.git
cd gcc
mkdir build-bootstrap
cd build-bootstrap
../configure --target=loongarch64-unknown-elf --without-headers --with-newlib --with-gnu-as --with-gnu-ld -enable-languages=c,c++
make all-gcc all-target-libgcc
make install-gcc install-target-libgcc
```

制作一个临时的 GCC 编译器，为之后编译 Newlib 做准备。

## Newlib

Newlib 需要添加 LoongArch 的支持，改动相对较多，但比较简单。

对于 RT 来说，我们暂时不用考虑数学库 `libm` 的功能，只需要在 `libgloss` 里添加一些系统调用（甚至可以不添加，直接用 libnosys），以及在 `libc` 里添加 `setjmp` 相关的实现，最后更新一下 `configure` 和 `Makefile.in` 等文件。

我这里依然基于了主线修改，主线的 commit 号是 `e4cc9e48462b538253d62109012b90befaaf7bc5`。

```bash
git clone https://github.com/FreeFlyingSheep/newlib.git
mkdir newlib-loongarch64-elf
cd newlib-loongarch64-elf
../newlib/configure --target=loongarch64-unknown-elf
make
make install
```

## GCC

现在我们有了 Newlib 的支持，可以制作一个完整的 GCC 了。

因为这里我只需要用来编译 RT，所以只启用了 C 和 C++ 的支持。
仓库仍然使用上面那个打了补丁的。

```bash
cd gcc
mkdir build
cd build
../configure --target=loongarch64-unknown-elf --with-newlib --with-gnu-as --with-gnu-ld --enable-languages=c,c++ --disable-shared
make
make install
```

## RT-Thread

RT-Thread 移植到 LoongArch64 的过程和 LoongArch32 几乎完全相同，区别是在原先的基础上新加了虚拟机 I/O 中断控制器的初始化，简化了 UART 驱动和中断/异常处理的代码。

这里我提供了 QEMU/LoongArch64 Virt 的板级支持（`bsp` 里添加 `qemu-virt64-loongarch64` 目录，`libcpu` 里添加 `loongarch` 目录）。

我还是基于主线进行了移植，主线 commit 号是 `1b3d287ceef877ad298507acaa081677936abe01`。

```bash
git clone https://github.com/FreeFlyingSheep/rt-thread.git
cd rt-thread/bsp/qemu-virt64-loongarch64
scons
```

## QEMU

QEMU 推荐使用 7.2 以后的版本，LoongArch 的支持已经比较完善了。

在刚刚的目录（`rt-thread/bsp/qemu-virt64-loongarch64`）直接执行 `qemu.sh` 即可在虚拟机里运行 RT-Thread。

现在我们就可以愉快地在虚拟机里调试 RT-Thread 了（执行 `qemu-dbg.sh`，配合 `loongarch64-unknown-elf-gdb`）。
