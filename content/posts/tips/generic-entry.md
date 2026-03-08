---
title: "Linux 中断和系统调用通用框架"
date: 2023-01-30
description: ""
menu:
  sidebar:
    name: "Linux 中断和系统调用通用框架"
    identifier: tips-generic-entry
    parent: tips
    weight: 100
tags: ["Linux 内核", "中断", "系统调用"]
categories: ["Tips"]
---

介绍 Linux 内核中断和系统调用部分的代码。

<!--more-->

## 通用框架的引入

在通用框架引入前，Linux 各个架构的中断处理和系统调用入口都是每个架构自己实现的，而且大部分是汇编语言实现的。
这就让这部分代码更难阅读、维护。

实际上不同架构的这部分代码逻辑相似，而且大部分逻辑可以用 C 去实现。

终于，在 Linux 5.8-rc6 的时候，有大佬开始根据 x86 的逻辑，实现中断和系统调用的通用框架，具体可以查看 `kernel/entry/common.c` 的 git log。

在通用框架完善后，其他架构如 s390、loongarch（loongarch 因为是新架构，进入主线的时候就已经用了通用框架）都积极跟进，现在使用通用框架是一个趋势。

## 新旧代码的对比

我曾经试图为 MIPS 使用通用框架，但最终因为影响性能的关系，遭到一部分人的反对，所以相关的补丁并没有继续推进，比较遗憾。
链接在这里：<https://lore.kernel.org/linux-mips/cover.1634177547.git.chenfeiyang@loongson.cn/T/#u>。

下面我就以 MIPS 为例，介绍如何把旧的代码重构为使用通用框架的。

### 系统调用

通用框架将 trace、seccomp、ptrace 以及处理线程标志的内容全部整合了。

`syscall_enter_from_user_mode()` 函数最终会调用 `syscall_trace_enter()`，该函数依次处理 Syscall User Dispatch、ptrace 和 seccomp。

`syscall_exit_to_user_mode()` 函数最终会调用 `exit_to_user_mode_loop()`，该函数会处理各种线程信息的标志（thread information flags）。
为此，我们需要在 `struct thread_info` 里添加/修改 `syscall` 和 `syscall_work` 两个成员，同时可能需要修改 `asm/thread_info.h` 中的标志。

这部分内容原本都是各架构用汇编实现的，现在我们可以直接交给通用框架：在进入系统调用时调用 `syscall_enter_from_user_mode()`，然后实际执行系统调用，退出前调用 `syscall_exit_to_user_mode()`。

MIPS 的系统调用可谓非常繁琐，因为存在着 O32、N32 和 N64 三种 ABI，然后结合 32 位和 64 位系统，所以设计了四个不同的入口：

- 32 位系统下的 O32（`arch/mips/kernel/scall32-o32.S`）
- 64 位系统下的 N32（`arch/mips/kernel/scall64-n32.S`）
- 64 位系统下的 N64（`arch/mips/kernel/scall64-n64.S`）
- 64 位系统下的 O32（`arch/mips/kernel/scall64-o32.S`）

仔细观察这四个汇编文件，发现他们的区别仅在于系统调用号和取系统调用参数的方式，完全可以将它们合并，并将核心逻辑用 C 语言重写。

我们保留一个调用入口 `handle_sys()`，在这里首先保存必要的寄存器、关中断，之后交给 `do_syscall()` 函数，该函数返回后恢复寄存器并回到用户态。
作为主体的 `do_syscall()` 是用 C 编写的，这样，我们就让汇编需要处理的部分尽可能少，大大增强可读性。

我们重构后的 `do_syscall()` 核心逻辑如下：

1. 调用 `syscall_enter_from_user_mode()`。
2. 给 `epc` 寄存器加 4，确保返回用户态时是跳到下一条指令。
3. 用 26 号寄存器保存系统调用号，用于系统调用重启。
4. 清空 27 号寄存器，用于标记是否需要直接返回（这里是为了性能优化，详细后面会说）。
5. 依次判断是否是 O32、N32、N64 系统调用。
6. 实际执行系统调用（调用相应的内核函数）。
7. 如果 27 号寄存器被置位，直接返回 `handle_sys()`。
8. 对系统调用返回值进行处理（MIPS 使用 7 号寄存器来保存错误标志，需要特别处理一下）。
9. 调用 `syscall_exit_to_user_mode()`。
10. 返回 `handle_sys()`。

这里涉及两个细节：获取系统调用参数和性能优化。

先介绍一下如何获取系统调用参数。
我们把四个入口合并了，必须保证逻辑和原始代码等价，主要是需要对 O32 进行特殊处理：

- 要对寄存器里的参数进行符号扩展，确保在 64 位下不丢失符号。
- 寄存器里获取 4 个参数，另外尝试从栈里获取另外 4 个参数。

之后介绍性能优化（负优化，逃。
在未使用通用框架前，我们注意到 `fork()`、`clone()`、`clone3()` 和 `sysmips()` 四个系统调用会使用 `save_static_function()` 来保存一些寄存器，而现在我们总是在 `handle_sys()` 里使用 `SAVE_STATIC`，因此 `save_static_function()` 已经不需要了。
这就是负优化的地方，我们增加了其他系统调用 `SAVE_STATIC` 的开销。
实际这个问题基本只在 MIPS 上存在，比如 RISC-V 一开始就是保存恢复所有寄存器的。
除此之外，因为 `fork()`、`clone()` 等系统调用会有两个返回路径，我们需要在另一个路径 `ret_from_kernel_thread()` 和 `ret_from_fork()` 里，也调用 `syscall_exit_to_user_mode()` 并恢复必要的寄存器，确保一致。

我们还注意到 `sigreturn()`、`rt_sigreturn()` 和 `sysmips()` 系统调用使用了内联汇编来直接返回，并标上了醒目的注释 `Don't let your children do this ...`， 那我们当然要干掉这个（逃
研究这几个函数的调用流程后，我发现内联汇编直接返回有两个用处：

- 略去对返回值的处理，不需要设置错误标志了。
- 不需要恢复部分寄存器。

于是，我们可以利用 27 号寄存器，来作为是否需要直接返回的标志，并在 `do_syscall()` 和 `handle_sys()` 中做特殊处理，就像上面说的那样。
略去对返回值的处理是必要的，但不需要恢复部分寄存器是为了性能优化。

重构后系统调用后，基本上只多了部分系统调用 `SAVE_STATIC` 的开销，其他逻辑是与之前等价的。

### 中断/异常

通用框架将 trace 和 RCU 相关的代码整合了，我们只需要在中断处理函数中成对地调用 `irqentry_enter()` 和 `irqentry_exit()` 函数。

MIPS 的中断处理函数由 `BUILD_HANDLER` 宏生成，旧实现会在汇编代码开关中断，而新实现我们必须把开中断放在 `irqentry_enter()` 和 `irqentry_exit()` 之间。

因此我们的异常处理函数核心逻辑如下：

1. 保存所有寄存器。
2. 关中断。
3. 调用 `irqentry_enter()`。
4. 需要的话开中断。
5. 执行异常处理。
6. 关中断。
7. 调用 `irqentry_exit()`。
8. 恢复所有寄存器并返回。

中断处理函数的核心逻辑如下：

1. 保存所有寄存器。
2. 关中断。
3. 调用 `irqentry_enter()`。
4. 调用 `set_irq_regs()` 保存任务现场。
5. 需要的话切换中断栈（用内联汇编来实现）。
6. 调用 `set_irq_regs()` 恢复任务现场。
7. 调用 `irqentry_exit()`。
8. 恢复所有寄存器并返回。

重构中断处理后，我们开中断的时机比原先要晚一些，这可能会造成性能损耗。

## 小结

中断和系统调用通用框架易于理解和维护，尽可能地分离了架构相关和架构无关的代码，是一个趋势。

但对于类似 MIPS 的架构，改用通用框架后不可避免地会造成性能损耗，这对于一些低性能的芯片是比较致命的。

很遗憾花了大量时间重构 MIPS 中断和系统调用的代码，没能进入上游，不过经过这次实践，确实对相关内容理解更透彻了。
