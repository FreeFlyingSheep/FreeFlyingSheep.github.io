---
title: "Valgrind 简介"
date: 2021-12-07
description: ""
menu:
  sidebar:
    name: "Valgrind 简介"
    identifier: valgrind-valgrind
    parent: valgrind
    weight: 100
tags: ["Valgrind"]
categories: ["Valgrind"]
---

简单介绍 Valgrind 及源码。

<!--more-->

## Valgrind 介绍

这部分内容主要来源于 Valgrind 官网的[用户手册](https://valgrind.org/docs/manual/manual.html)。

[Valgrind](https://valgrind.org/) 是一个用于建立动态分析工具的插桩（Instrumentation）框架。
其本质是一个类似 QEMU TCG 的虚拟机（似乎 Valgrind 比 QEMU 项目更早创立），在动态翻译的过程中可以插入很多检查，同时提供了一个 gdbserver 方便远程调试。

### 工具介绍

- Memcheck 是一个内存错误检测器。它可以帮助你使你的程序更加正确，特别是那些用 C 和 C++ 编写的程序。
- Cachegrind 是一个高速缓存和分支预测分析器。它能帮助你使你的程序运行得更快。
- Callgrind 是一个生成调用图的缓存分析器。它与 Cachegrind 有一些重叠，但也收集了一些 Cachegrind 所没有的信息。
- Helgrind 是一个线程错误检测器。它可以帮助你使你的多线程程序更加正确。
- DRD 也是一个线程错误检测器。它与 Helgrind 相似，但使用不同的分析技术，因此可能会发现不同的问题。
- Massif 是一个堆分析器。它可以帮助你使你的程序使用更少的内存。
- DHAT 是一个不同类型的堆分析器。它可以帮助你了解区块寿命、区块利用率和布局效率低下的问题。
- BBV 是一个实验性的 SimPoint 基本块向量发生器。它对从事计算机结构研究和开发的人很有用。
- Lackey 是一个例子工具。它说明了一些插桩的基本原理。
- Nulgrind 是最小的 Valgrind 工具。它不做任何分析或插桩，只对测试有用。

### 简单使用

Valgrind 必须安装到编译时指定的安装目录才能正常使用。

对我来说，我只需要尽可能多地输出信息来帮助我理解、调试（逃。因此常用的参数有下面几个：

- `-h, --help`：显示帮助信息。
- `--help-debug`：显示调试相关的帮助信息。
- `--version`：显示版本信息。
- `-v, --verbose`：在运行程序时输出更多的信息。
- `-d`：输出更多调试信息。
- `--tool=<toolname>`：指定工具，对应上面的[工具介绍](#工具介绍)，分别是 `memcheck`、`cachegrind`、`callgrind`、`helgrind`、`drd`、`massif`、`dhat`、`exp-bbv`、`lackey` 和 `none`。不指定的话默认 `--tool=memcheck`。
- `--log-file=<filename>`：指定日志文件。
- `--trace-syscalls=no|yes`：追踪系统调用。
- `--trace-notbelow=<number>, --trace-notabove=<number>`：只输出不低于/不高于指定级别的调试信息。
- `--trace-flags=<XXXXXXXX>`：打开/关闭追踪标志，与上面的选项结合来输出更多调试信息。

举个例子，这么使用 Valgrind 运行 `hello` 程序能打印足够多的信息给我调试（Valgrind 默认安装位置是 `/usr/local`）：

```bash
/usr/local/bin/valgrind -v -d --tool=memcheck --trace-syscalls=yes --trace-flags=11111111 --trace-notbelow=0 --log-file=./log.txt ./hello
```

输出信息的整体可读性比较强，最左侧 `==12345==` 之类的标识符是当前的进程 ID。

### 编译安装

和大部分软件编译安装的一样，如果是拉源码仓库编译安装就依次执行下面的命令：

```bash
git clone git://sourceware.org/git/valgrind.git
cd valgrind
./autogen.sh
./configure --prefix=...
make
make install
```

## Valgrind 源码

个人感觉 Valgrind 源码比较乱。可能是该项目最初设计的时候没有约定代码风格，也没有很好地考虑多架构的支持，导致经常在同一个文件中混杂大段的来自不同架构的代码（甚至还是不同代码风格的），存在着不少需要十几个参数的函数，还有就是各种神奇的缩写。

因为我学习 Valgrind 的最终目标是[适配 LoongArch](/zh-cn/posts/valgrind/loongarch)，所以后面仅针对 Linux 平台，关注整体流程和架构相关的部分。

### 常见缩写

- `VG_*`: Valgrind（很多函数名被包在 `VG()` 中，这是一个宏，通常被展开为 `vgPlain_`，这么做或许是为了避免函数名冲突）
- `ML_*`: Module（`ML()` 宏被展开为 `vgModuleLocal_`，和 `VG()` 差不多，都是给函数名添加前缀）
- `VGA_*/VGO_*/VGP_*`: Valgrind arch/OS/platform-related variables
- `vki_*`: Valgrind kernel interface
- `SSA`: Static Single Assignment
- `IR`: Intermediate Representation
- `BB`: Basic Block
- `IRSB`: IR Super Block
- `ISEL`: Instruction selector
- `AMODEs`: Addressing modes
- `SCSS`: Static client signal state（指 `static SCSS scss;` 静态变量）
- `SKSS`: Static kernel signal state（指 `static SKSS skss;` 静态变量）
- `DWARF`: Debugging with attributed record formats
- `CFA`: Canonical Frame Address
- `CFI`: Call Frame Instructions
- `*_WRK`/`*_wrk`: `RK` 代表 do the real work，`W` 推测是 wrapper（经常作为函数后缀）
- `pp*`: Pretty print（经常作为打印用的函数的前缀）
- `NB`: Nota Bene（经常在注释里看到，来自于拉丁文，“注意”的意思）
- `ASAP`: As Soon As Possible（经常在注释里看到）

特此声明，arch/OS/platform 指架构（体系结构）/操作系统/平台，在 Valgrind 中，platform 是 arch + OS，而在后文中因为我默认“操作系统”是 Linux，所以经常混用“架构”和“平台”。
后文大写的 `XXX` 指代任意指架构名，如 `X86`、`MIPS64`。

### 启动

启动部分的代码主要位于 `coregrind/m_main.c`。

因为 Valgrind 并不会链接 glibc，所以程序的入口是 `_start` 而不是 `main()` 函数。

在 `_start` 处，我们需要先用汇编语言来分配供 C 语言使用的栈空间，然后跳转到 C 语言的开始函数 `_start_in_C_linux()`，显然这部分是架构相关的。

`_start_in_C_linux()` 函数会做一些基本的初始化工作：

1. 计算 `argc`、`argv` 和 `envp` 的地址。
2. 注册临时栈（看注释似乎是用于 Valgrind 嵌套执行 Valgrind 的时候）。
3. 对于可以配置页大小的架构，解析（ELF interpreter info）来设置正确的页大小和偏移量。
4. 调用真正的主函数 `valgrind_main()`。

`valgrind_main()` 主要干了以下的工作（这里基本就是翻译了一下注释）：

1. 解析命令行参数中日志相关的部分，启动日志机制（logging mechanism）。
2. 开启[调试记录器](#调试记录debug-logging)（debug logger）。
3. 检查当前栈的合理性。
4. 检查初始栈的合理性（初始栈是给在 Valgrind 中运行的客户端程序用的）。
5. 启动地[址空间管理器](#地址空间管理器address-space-manager)（address space manager）。
6. 启动动态内存管理器（dynamic memory manager）。
7. 初始化[调试信息](#调试信息debug-info)（debug info）。
8. 通过环境变量，查找是否有替代的库目录。
9. 通过环境变量，检查启动器的名称。
10. 通过 `getrlimit()` [系统调用](#系统调用syscall)获取一些限制信息（`RLIMIT_DATA` 和 `RLIMIT_STACK`）。
11. 获取 CPU 硬件信息（包括了 CPU 高级[缓存](#缓存cache)的信息）。
12. 获取工作目录。
13. 分割命令行参数（主要是分割 Valgrind 的参数和客户端程序的参数）。
14. 解析 Valgrind 工具的部分参数。
15. 设置 libVex 默认参数。
16. 加载客户端程序，包括建立运行环境、建立栈、建立堆等，以上工作也被称为建立[初始镜像](#初始镜像initial-image)（create initial image）。
17. 建立文件描述符。
18. 创建伪造的 `/proc/<pid>/cmdline`。
19. 创建伪造的 `/proc/<pid>/auxv`。
20. Valgrind 工具的初始化第一阶段，解析 Valgrind 和相关工具的参数。
21. Valgrind 工具的初始化第二阶段，进行一些检查工作。
22. 初始化翻译表（translation table）和翻译缓存（translation cache）。
23. 初始化[重定向](#重定向截取redirectionintercept)表（redirect table）。
24. 查找从父进程继承的文件描述符。
25. 为存在的段加载调试信息。
26. 通知地址空间管理器（aspacem）汇编帮助函数的拥有权的改变，初始化线程状态（thread state）模块。
27. 初始化调度器第一阶段，包括初始化 CPU 锁（bigLock），清零 `VG_(threads)` 结构体，以及决定根线程（root thread）的线程号。
28. 通知 Valgrind 工具内存的权限。
29. 初始化调度器第二阶段，包括填充栈的具体信息，获取 CPU 锁，以及正式初始化调度器。
30. 建立根线程的状态，其中包括了一个步骤叫最终确立初始镜像（finalise initial image）。
31. 初始化[信号处理系统](#信号signal)（signal handling subsystem）。
32. 读取压制（suppression）文件。
33. 注册客户端程序的栈。
34. 展示目前的地址空间。
35. 正式运行[主线程](#主线程)（main thread）。

特地强调“建立初始镜像”和“最终确立初始镜像”是因为这两个的缩写实在太魔性了：

- `iicii`: initial image, create initial image
- `iifii`: initial image, finalise initial image

### 调试记录（debug logging）

调试记录模块的代码位于 `coregrind/pub_core_debuglog.h` 和 `coregrind/m_debuglog.c`。

注意，调试记录模块是用于记录 Valgrind 本身的日志的，而不是记录客户端程序的日志。

`VG_(debugLog_startup)()` 函数负责启动调试记录器，该函数只干了一件事：设置调试日志级别。

`VG_(debugLog)()` 函数负责发送调试输出，期间要获取进程 ID（PID）。
获取 PID 和输出到 `stderr` 的过程是架构相关的，分别使用汇编语言进行 `getpid()` 和 `write()` 系统调用。

### 地址空间管理器（address space manager）

地址空间管理器模块的代码位于 `coregrind/pub_core_aspacehl.h`、`coregrind/pub_core_aspacemgr.h` 和 `coregrind/m_aspacemgr` 目录。

`VG_(am_startup)()` 函数负责初始化地址空间管理器，在 64 位系统上最终的内存布局如下：

```text
,--------------------------------, 0x00000000_00000000
|                                |
|--------------------------------| 0x00000000_00400000
|          client text           |
|--------------------------------|
|                                |
|                                |
|--------------------------------|
|          client stack          |
|--------------------------------| 0x00000000_58000000
|            V's text            |
|--------------------------------|
|                                |
|--------------------------------|
|     dynamic shared objects     |
|--------------------------------| 0x0000001f_ffffffff
|                                |
|                                |
|--------------------------------|
| initial stack given to V by OS |
'--------------------------------' 0xffffffff_ffffffff
```

这部分内容涉及的系统调用主要有 `mmap()`、`open()`、`close()`、`read()`、`readlink()` 和 `fcntl()`，因为高版本的内核可能提供了新系统调用，所以各个架构需要根据自身情况在相应的封装函数里选择正确的代码块（`#if defined(VGA_XXX_linux)`）。

### 调试信息（debug info）

调试信息模块的代码位于 `coregrind/pub_core_debuginfo.h` 和 `coregrind/m_debuginfo` 目录。

Valgrind 支持 [DWARF5](https://dwarfstd.org/doc/DWARF5.pdf)。

对于特定的架构，基本上只需要在对应的地方填入相应的栈指针寄存器、帧指针寄存器、函数返回寄存器等，剩下的交给公共框架就行。

### 系统调用（syscall）

系统调用相关的有两个模块：系统调用（syscall）模块和系统调用包装器（syscall wrapper）模块。

前者的代码位于 `coregrind/pub_core_syscall.h` 和 `coregrind/m_syscall.c`，负责在对应的平台上实际执行系统调用。
后者的代码位于 `coregrind/pub_core_syswrap.h` 和 `coregrind/m_syswrap` 目录，包含了大量的辅助函数和宏。

系统调用模块定义的“实际执行系统调用”的函数是 `VG_(do_syscall)()`，它主要干了两件事情：

1. 调用系统调用封装函数 `do_syscall_WRK()` 来真正执行系统调用，这个函数基本是汇编实现的。
2. 调用 `mk_SysRes_XXX_linux()` 函数，该函数把系统调用返回值转换成 `SysRes` 结构体。

一些模块自身也要使用系统调用，它们往往又在 `VG_(do_syscall)()` 基础上封装了一层。
比如 `VG_(open)()` 函数封装了 `open()` 和 `openat()` 系统调用，不同平台要选择调用哪个。
所谓的“在封装函数里选择正确的代码块”，就是针对这些函数。
因为是需要用到时模块自己定义的，所以这些封装函数散布在各个地方，个人感觉非常杂乱。

每个架构需要给出自己的系统调用表，系统调用包装器模块提供了几个有用的宏（不难发现其中的规律）：

- `GENX_()`：表示该系统调用是全平台的，在调用前需要进行额外处理
- `GENXY()`：表示该系统调用是全平台的，在调用前和后都需要进行额外处理
- `LINX_()`：表示该系统调用是 Linux 平台的，在调用前需要进行额外处理
- `LINXY()`：表示该系统调用是 Linux 平台的，在调用前和后需要进行额外处理

额外处理即调用封装函数，这些函数通常会根据系统调用的参数/返回值来跟踪一些读写操作。
注意，调用前的额外处理是永远需要的，因为总要追踪读取系统调用号的操作。

对于不能走公共代码的架构相关的系统调用，我们需要自定义调用前和后的处理函数。

具体来说，如果 `XXX` 平台有个独一无二的系统调用 `sys_xxx`，那么我们可以这样写：

```c
#define PRE(name)  DEFN_PRE_TEMPLATE(XXX, name)
#define POST(name) DEFN_POST_TEMPLATE(XXX, name)

DECL_TEMPLATE(XXX, sys_xxx);

PRE(sys_xxx)
{
    // 调用前的额外处理
}

POST(sys_xxx)
{
    // 调用后的额外处理
}

#define PLAX_(sysno, name) WRAPPER_ENTRY_X_(XXX, sysno, name)
#define PLAXY(sysno, name) WRAPPER_ENTRY_XY(XXX, sysno, name)

// 系统调用表
static SyscallTableEntry syscall_main_table[] = {
    // 其他系统调用
    PLAXY(__NR_XXX, sys_xxx),
    // 其他系统调用
};
```

其中，`__NR_XXX` 是该系统调用的系统调用号，来自内核。
而 `DEFN_PRE_TEMPLATE()`、`DEFN_POST_TEMPLATE()`、`DECL_TEMPLATE()`、`WRAPPER_ENTRY_X_()` 和 `WRAPPER_ENTRY_XY()` 都是系统调用包装器模块提供的辅助宏，看字面意思就能理解（DEFN 是 define，DECL 是 declare）。

### 缓存（cache）

缓存相关的代码位于 `coregrind/m_cache.c` 和 `cachegrind` 目录。

`coregrind/m_cache.c` 主要就是探测缓存信息，因为不同 CPU 的探测方式都不同，所以每个架构都实现了一套。

cachegrind 是 Valgrind 的缓存工具，里面基本都是公共化的代码，基本上每个架构只需要设置一些默认值。

### 初始镜像（initial image）

初始镜像模块的代码位于 `coregrind/pub_core_initimg.h` 和 `coregrind/m_initimg` 目录。

该模块负责把客户端可执行文件映射到内存，完成执行前到准备工作：建立栈、环境、数据段（堆）。

因为遵循 ELF 规范，所以大部分代码都是公共的，架构相关的只有初始化 VEX 状态（`VexGuestXXXState`）。

### 信号（signal）

信号相关的也有两个模块：信号（signal）模块和信号栈（signal frame）模块。

前者的代码位于 `coregrind/pub_core_signals.h` 和 `coregrind/m_signals.c`，负责信号处理。
后者的代码位于 `coregrind/pub_core_sigframe.h` 和 `coregrind/m_sigframe` 目录，负责为客户端线程创建/销毁信号栈（包括保存/恢复 CPU 状态）。

`VG_(sigstartup_actions)()` 函数负责启动信号处理系统。

信号栈的创建和销毁显然是架构相关的，代码位于 `coregrind/m_sigframe/sigframe-XXX-linux.c`。

### 重定向/截取（redirection/intercept）

Valgrind 的重定向/截取模块的代码位于 `coregrind/m_redir.c`。

该模块负责跟踪当前的截取状态，当状态改变时清理翻译缓存，以及判断地址是否被重定向。

`VG_(redir_initialise)()` 函数负责初始化重定向系统，每个架构需要指定自己的 `ld.so` 文件。

### 主线程

`valgrind_main()` 最后通过调用 `VG_(main_thread_wrapper_NORETURN)()` 来运行主线程。

该函数会分配一个新的栈，然后在新栈上执行 `run_a_thread_NORETURN()`，分配栈以及切栈的过程是架构相关的。

`run_a_thread_NORETURN()` 获取线程状态、注册 Valgrind 自身使用的栈，然后调用 `thread_wrapper()` 函数。

`thread_wrapper()` 获取 CPU 锁，设置线程状态，然后调用 `VG_(scheduler)()` 把控制权交给[调度器](#调度器scheduler)。

从调度器返回后回到 `thread_wrapper()`，然后返回 `run_a_thread_NORETURN()`，后者依次取消注册线程栈、更新线程状态、调用 `exit()` 系统调用退出。

除了 `VG_(scheduler)()` 位于 `coregrind/m_scheduler/scheduler.c`，其他几个函数都位于 `coregrind/m_syswrap/syswrap-linux.c`。

### 调度器（scheduler）

`VG_(scheduler)()` 会进行以下操作：

1. 获取线程状态。
2. 对于首次运行的主线程，初始化 [vgdb](#gdbserver)。
3. 设置栈的缓存（stack cache）。
4. 设置信号掩码（signal mask）。
5. 设置该线程的时间片的剩余时间，执行线程直到线程退出。

执行线程这一步是在一个超大的循环中完成的。

当时间片归零时：

1. 释放 CPU 锁（这会让出 CPU 执行权，具体细节见 `coregrind/m_scheduler/ticket-lock-linux.c`）。
2. 再次获取 CPU 锁。
3. 进行一些检查工作。
4. 传递（deliver）信号（若有的话）。
5. 若线程要退出，退出循环。
6. 更新统计数据（`n_scheduling_events_MAJOR`）。
7. 更新时间片。

当时间片大于零时：

1. 更新统计数据（`n_scheduling_events_MAJOR`）。
2. 调用 `run_thread_for_a_while()` 执行线程。
3. 根据 `run_thread_for_a_while()` 的返回值，进行相应的处理，这部分和 [libVEX](#libvex) 息息相关。

`run_thread_for_a_while()` 会调用 `VG_(disp_run_translations)()` 进行[分派](#分派dispatch)。

### gdbserver

gdbserver 模块的代码位于 `coregrind/pub_core_gdbserver.h` 和 `coregrind/m_gdbserver` 目录。

该模块基本是根据 gdb 6.6 的相关代码改造的。

### libVEX

libVEX 的代码位于 `VEX` 目录。

libVEX 库是 Valgrind 的重要组成部分，简单地说，libVEX 的前端负责反汇编（disassemble）二进制指令到 Vex IR 语言，后端则根据 Vex IR 语言发射（emit）二进制指令。

为了能分析和执行不同架构的代码，Valgrind 会利用 libVEX 库将所有架构的二进制指令流先翻译成 Vex IR，最后交给 CPU 执行时再翻译回二进制。

翻译（translate）模块、调度器对 `run_thread_for_a_while()` 返回值的处理以及 libVEX 的更多内容见 [libVEX 简介](/zh-cn/posts/valgrind/libvex)。

### 分派（dispatch）

分派模块的代码位于 `coregrind/pub_core_dispatch_asm.h`、`coregrind/pub_core_dispatch.h` 和 `coregrind/m_dispatch` 目录。

该模块负责实现 Valgrind 内循环的执行机制：找到下一个基本块并执行，重复直到以下三种情况发生：

- 下一个基本块在快速缓存（fast-cache）中找不到
- 当前的基本块退出并请求一些特殊的操作
- 当前线程用完了它的调度时间片

`VG_(disp_run_translations)()` 函数在 `coregrind/m_dispatch/dispatch-XXX-linux.S` 中定义，它会分配新栈并将控制权转移给指定的代码块。

该函数通过指针来返回数据，返回值叫 `two_words`，指会返回两个 `HWord`（halfword，被定义为 `unsigned long`）类型的变量。
第一个 `HWord` 被称为 `TRC`（鬼知道是什么的缩写），它代表着一些调度事件，用来指示调度器进行后续的操作。
第二个 `HWord` 不常使用，只在 chain-me 请求的时候会用到。

chain-me 请求指动态翻译时已经能确定目的地的那些情况，一共有两个函数对应 chain-me 请求：`VG_(disp_cp_chain_me_to_slowEP)()` 和 `VG_(disp_cp_chain_me_to_fastEP)()`。
这两个函数也在 `coregrind/m_dispatch/dispatch-XXX-linux.S` 中定义，它们会在栈中预留出加载函数地址的指令和跳转指令的空间，供 libVEX 来给自己“打补丁”（patch），打补丁的操作由 [transtab](#transtab) 模块完成。

### transtab

transtab 模块的的代码位于 `coregrind/pub_core_transtab_asm.h`、`coregrind/pub_core_transtab.h` 和 `coregrind/m_transtab.c`。

该模块负责缓存翻译和快速查找缓存。

这里着重关注一下上文提到的打补丁的流程：

1. `VG_(scheduler)()` 函数在处理 `run_thread_for_a_while()` 的返回值时，遇到 `VG_TRC_CHAIN_ME_TO_SLOW_EP` 和 `VG_TRC_CHAIN_ME_TO_FAST_EP`，进而调用 `handle_chain_me()` 函数。
2. `handle_chain_me()` 调用 transtab 模块的 `VG_(tt_tc_do_chaining)()` 函数。
3. `VG_(tt_tc_do_chaining)()` 调用 libVEX 库的 `LibVEX_Chain()` 函数，同时根据情况传入 `VG_(disp_cp_chain_me_to_slowEP)()` 或 `VG_(disp_cp_chain_me_to_fastEP)()`。
4. `LibVEX_Chain()` 函数调用架构相关的 `chainXDirect_XXX()` 函数。
5. `chainXDirect_XXX()` 获取第 3 步传入的函数地址，在对应地方填入加载函数地址的指令和跳转指令。

### 跳版（trampoline）

跳版部分的代码位于 `coregrind/m_trampoline.S`。

该部分负责实现一些在模拟 CPU 上执行的帮助函数，例如 `index()`、`strlen()`、`sigreturn()` 等函数。

正如注释说的，不知道为什么要用汇编来实现这些函数，但既然其他架构都这么干了，就只能继续下去（无力吐槽）。

### 内存检查（memcheck）

memcheck 工具的代码位于 `memcheck` 目录，该工具负责内存检查相关的工作。

各个架构需要在 `memcheck/mc_machine.c` 的 `get_otrack_shadow_offset_wrk()` 函数里，填入各自客户机状态结构体（`VexGuestXXXState`）成员的偏移量（没被跟踪的成员返回 `-1`）。

### 栈回溯（stacktrace）

栈回溯模块的代码位于 `coregrind/pub_core_stacktrace.h` 和 `coregrind/m_stacktrace.c`。

该模块负责 unwind 栈回溯，各个架构需要实现自己的 `VG_(get_StackTrace_wrk)()` 函数。

### 核心转储（core dump）

核心转储模块的代码位于 `coregrind/pub_core_coredump.h` 和 `coregrind/m_coredump` 目录。

该模块提供核心转储功能，各个架构需要分别在 `coregrind/m_coredump/coredump-elf.c` 的 `fill_prstatus()` 和 `fill_fpu()` 函数中填入自己的整数和浮点寄存器。

### Valgrind 内核接口（Valgrind kernel interface）

Valgrind 内核接口的代码位于 `coregrind/pub_core_vki.h`、`coregrind/pub_core_vkiscnums_asm.h`、`coregrind/pub_core_vkiscnums.h` 和 `include/vki` 目录。

该模块的代码基本来源于内核 2.6.x 和 3.x 的相关头文件，并在所有类型名、宏名前加上 `VKI_`/`vki_`，这么做是为了引用内核对外隐藏的一些数据结构。

每个架构需要提供三个文件：

- `include/vki/vki-posixtypes-XXX-linux.h`: 源自内核 `posix_types.h` 文件。
- `include/vki/vki-scnums-XXX-linux.h`: 源自内核系统调用号头文件。
- `include/vki/vki-XXX-linux.h`: 源自内核其他头文件，包含所有其他需要的类型和宏（所以看上去很杂乱）。

### 其他

剩下的部分里，架构相关的比较少，基本都是在对应的地方选择默认配置、代码块（`#if defined(VGA_XXX_linux)`），就能正常工作了。
