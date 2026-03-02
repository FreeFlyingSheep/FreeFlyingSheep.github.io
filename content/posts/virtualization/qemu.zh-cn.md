---
title: "QEMU 简介"
date: 2021-06-08
description: ""
menu:
  sidebar:
    name: "QEMU 简介"
    identifier: virtualization-qemu
    parent: virtualization
    weight: 100
tags: ["QEMU"]
categories: ["Virtualization"]
---

根据 [QEMU 官方文档](https://qemu-project.gitlab.io/qemu/index.html) 和 [Qemu Detailed Study](https://lists.gnu.org/archive/html/qemu-devel/2011-04/pdfhC5rVdz7U8.pdf) 整理。

<!--more-->

## 相关概念介绍

### QEMU

QEMU 是一个通用的、开源的机器仿真器和虚拟机。

在系统模式（System mode）下，QEMU 能运行各种不同架构的操作系统。

在用户模式（User mode）下，QEMU 能运行为其他架构编写的程序。

通常情况下，我们把 QEMU 要模拟的机器称为目标（Target）机器，运行 QEMU 的机器被被称为主机（Host）。

### TCG

以下部分内容来自[维基百科](https://en.wikipedia.org/wiki/Binary_translation)。

二进制翻译（Binary Translation）指把一种指令集重新编译成另一种指令集，即把源指令序列翻译为目标指令序列。

二进制翻译分为静态二进制翻译（Static binary translation）和动态二进制翻译（Dynamic binary translation，简称 DBT）两种。

前者指在执行程序前就将所有指令翻译好，这是很难正确做到的，因为不是所有的代码都能被翻译器发现（某些程序甚至会在运行过程中生成新的指令）；后者通常是执行时即时（Just in Time，简称 JIT）翻译简短的代码序列，然后将结果缓存起来，代码只在被发现和可能的情况下被翻译，并且分支指令会指向已经翻译和保存的代码。

TCG 是 QEMU 所采用的一种动态二进制翻译技术，全称是 Tiny Code Generator。

因为在 TCG 模式下，我们希望把源程序指令序列翻译成能在主机上运行的指令序列，所以 TCG 的 Target 是主机，这点一定要注意。

TCG 的翻译任务主要由两部分组成：第一部分是将目标代码块（翻译块，TB）转换为中间语言（TCG 操作，TCG ops）；第二部分是将 TB 的 TCG ops 转换为主机代码。

自然，转换过程中会伴随可选的优化过程。

### KVM

以下部分内容来自[维基百科](https://en.wikipedia.org/wiki/QEMU)。

KVM（基于内核的虚拟机）是一个 FreeBSD 和 Linux 内核的模块，它允许用户空间的程序访问各种处理器的硬件虚拟化功能。

当目标架构与主机架构相同时，QEMU 可以利用 KVM 的特殊功能，如加速功能。

## TCG 执行流程简介

代码基于 QEMU 6.0.0 版本。

### 程序入口

喜闻乐见的只有 5 行的 `int main(int argc, char **argv, char **envp)` 函数，位于 `softmmu/main.c`。

`void qemu_init(int argc, char **argv, char **envp)` 函数位于 `softmmu/vl.c`，该函数负责初始化工作，具体细节目前我并不关心。

大体概括一下，各个体系结构会用 `include/qom/object.h` 文件的 `DEFINE_TYPES(type_array)` 来定义 CPU 相关信息的结构体。

这个结构体包含了指向 `static void xxx_cpu_class_init(ObjectClass *oc, void *data)` 函数的函数指针，位于 `target/xxx/cpu.c` 中，其中 `xxx` 是具体的体系结构名，如 `i386`、`mips`。

`xxx_cpu_class_init()` 函数会调用 `static void xxx_cpu_realizefn(DeviceState *dev, Error **errp)` 函数。

而 `xxx_cpu_realizefn()` 函数最终会调用 `qemu_init_vcpu()` 函数来初始化虚拟 CPU。

### 创建虚拟 CPU

`void qemu_init_vcpu(CPUState *cpu)` 函数位于 `softmmu/cpus.c`，这里着重关心这个调用语句：`cpus_accel->create_vcpu_thread(cpu);`。

`create_vcpu_thread` 函数指针在 `static void tcg_accel_ops_init(AccelOpsClass *ops)` 中被初始化，该函数位于 `accel/tcg/tcg-accel-ops.c`。

`tcg_accel_ops_init` 函数指针在 `static void tcg_accel_ops_class_init(ObjectClass *oc, void *data)` 中被初始化。

这个函数指针在 `tcg_accel_ops_type` 结构体中，最终通过 `include/qemu/module.h` 中的 `type_init(function)` 宏来初始化。

回到 `create_vcpu_thread` 函数指针，该指针在多线程情况下被初始化为 `accel/tcg/tcg-accel-ops-mttcg.c` 下的 `void mttcg_start_vcpu_thread(CPUState *cpu)` 函数，单线程情况下则被初始化为 `accel/tcg/tcg-accel-ops-rr.c` 下的 `void rr_start_vcpu_thread(CPUState *cpu)` 函数。

`mttcg_start_vcpu_thread()` 函数会创建线程执行 `static void *mttcg_cpu_thread_fn(void *arg)`；`rr_start_vcpu_thread()` 函数会创建线程执行 `static void *rr_cpu_thread_fn(void *arg)` 。

不论是 `mttcg_cpu_thread_fn()` 还是 `rr_cpu_thread_fn()` 函数，最终都会调用 `int tcg_cpus_exec(CPUState *cpu)` 函数。

而 `tcg_cpus_exec()` 函数则会调用 `cpu_exec()` 函数执行代码。

### 动态翻译

`int cpu_exec(CPUState *cpu)` 函数位于 `accel/tcg/cpu-exec.c`，是 TCG 执行代码的主循环。

这里只考虑查找生成动态代码的部分，即调用的 `static inline TranslationBlock *tb_find(CPUState *cpu, TranslationBlock *last_tb, int tb_exit, uint32_t cflags)` 函数。

`tb_find()` 函数会调用 `accel/tcg/translate-all.c` 文件中的 `TranslationBlock *tb_gen_code(CPUState *cpu, target_ulong pc, target_ulong cs_base, uint32_t flags, int cflags)` 函数来生成动态代码。

`tb_gen_code()` 函数会调用体系结构相关的 `target/xxx/translate.c` 文件中的 `void gen_intermediate_code(CPUState *cpu, TranslationBlock *tb, int max_insns)` 函数来为生成中间代码做准备。

`gen_intermediate_code()` 函数调用 `accel/tcg/translator.c` 文件中的 `void translator_loop(const TranslatorOps *ops, DisasContextBase *db, CPUState *cpu, TranslationBlock *tb, int max_insns)`，该函数主体是体系结构无关的，但它又通过关联的函数指针调用体系结构相关的函数来执行翻译。

### 执行代码

完成翻译后，`cpu_exec()` 函数调用 `accel/tcg/cpu-exec.c` 文件中的 `static inline void cpu_loop_exec_tb(CPUState *cpu, TranslationBlock *tb, TranslationBlock **last_tb, int *tb_exit)` 函数，该函数会调用 `static inline TranslationBlock * QEMU_DISABLE_CFI
cpu_tb_exec(CPUState *cpu, TranslationBlock *itb, int *tb_exit)` 函数来执行动态翻译的代码。

之后的操作我暂时不关心，先看到这里为止。

## KVM 执行流程简介

QEMU KVM 模式下，QEMU 与内核 KVM 模块交互，这里只考虑 QEMU 部分，内核部分见 [KVM 简介](/zh-cn/posts/virtualization/kvm)。

KVM 的初始化流程大体和 TCG 部分介绍的差不多。

相关的函数和宏如下：

- `int main(int argc, char **argv, char **envp)`，位于 `softmmu/main.c`。
- `void qemu_init(int argc, char **argv, char **envp)`，`softmmu/vl.c`。
- `type_init(function)`，位于 `include/qemu/module.h`。
- `static void kvm_accel_ops_class_init(ObjectClass *oc, void *data)`，位于 `accel/kvm/kvm-accel-ops.c`。
- `static void kvm_start_vcpu_thread(CPUState *cpu)`，位于 `accel/kvm/kvm-accel-ops.c`。
- `static void *kvm_vcpu_thread_fn(void *arg)`，位于 `accel/kvm/kvm-accel-ops.c`。
- `int kvm_init_vcpu(CPUState *cpu, Error **errp)`，位于 `accel/kvm/kvm-all.c`。
- `int kvm_arch_init_vcpu(CPUState *cs)`，位于 `target/xxx/cpu.c`。
- `int kvm_cpu_exec(CPUState *cpu)`，位于 `accel/kvm/kvm-all.c`。

其中，`kvm_vcpu_thread_fn()` 函数先调用体系结构无关的初始化函数 `kvm_init_vcpu()`，再执行 `kvm_cpu_exec()` 函数来实际执行代码。

`kvm_init_vcpu()` 函数会先调用 `static int kvm_get_vcpu(KVMState *s, unsigned long vcpu_id)` 函数来与内核交互创建虚拟 CPU（通过 `int kvm_vcpu_ioctl(CPUState *cpu, int type, ...)` 函数与内核交互），然后调用体系结构相关的初始化函数 `kvm_arch_init_vcpu()`。

而 `kvm_cpu_exec()` 函数通过 `kvm_vcpu_ioctl()` 函数与内核交互，切换到 KVM 来实际执行代码。

具体细节和后续操作我暂时不关心，先鸽了。
