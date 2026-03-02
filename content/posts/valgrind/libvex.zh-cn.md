---
title: "libVEX 简介"
date: 2021-12-16
description: ""
menu:
  sidebar:
    name: "libVEX 简介"
    identifier: valgrind-libvex
    parent: valgrind
    weight: 100
tags: ["Valgrind", "libVEX"]
categories: ["Valgrind"]
---

简单介绍 Valgrind libVEX 及源码。

<!--more-->

## 编译原理相关知识

为了理解 libVEX 的很多概念，补习一下编译原理（逃。

以下内容来自《编译原理（第 2 版）》（龙书）第一章。

简单地说，一个编译器（compiler）就是一个程序，它可以阅读以某一种语言（源语言）编写的程序，并把该程序翻译成一个等价的、用另一种语言（目标语言）编写的程序。

把源程序映射为在语义上等价的目标程序的过程由两部分组成：分析（analysis）部分和综合（synthesis）部分。

分析部分把源程序分解成多个组成要素，并在这些要素之上加上语法结构。
然后，它使用这个结构来创建该源程序的一个中间表示。
如果分析部分检查出源程序没有按照正确的语法构成，或者语义上不一致，它就必须提供有用的信息，使得用户可以按此进行改正。
分析部分还会收集有关源程序的信息，并把信息放在一个成为符号表（symbol table）的数据结构中。
符号表将和中间表示形式一起传送给综合部分。

综合部分根据中间表示和符号表中的信息来构造用户期待的目标程序。

分析部分经常被称为编译器的前段（front end），而综合部分称为后端（back end）。

再进行细分的话，可以发现编译器顺序执行了一组步骤（phase）：

![一个编译器的各个步骤](/images/valgrind/compiler.png)

## libVEX 介绍

libVEX 库是 Valgrind 的核心，负责各架构的二进制代码和 Vex IR 语言之间的翻译，其代码位于 `VEX` 目录。

libVEX 包含了一个即时编译器（JIT），它的前端负责反汇编（disassemble）二进制指令到 Vex IR 语言，后端则根据 Vex IR 语言发射（emit）二进制指令。

除了 Valgrind，libVEX 库也被其他开源项目使用，比如 [angr](http://angr.io/)，angr 甚至有[文档](https://docs.angr.io/advanced-topics/ir)专门介绍 Vex IR。

## Vex IR

Vex IR（Vex Intermediate Representation）是一个架构无关的高级中间表示（high-level IR）。

Vex IR 的介绍和类型定义基本都放在 `VEX/pub/libvex_ir.h` 里，并带有相当多的注释（因为注释已经够完善了，所以就不需要文档了？）。

注意，后文贴的代码重新排版过，直接搜索可能搜不到。

### 客户机状态（guest state）

运行 libVEX 的机器称为宿主机（host），libVEX 模拟运行的机器称为客户机（guset）。

客户机状态包含了客户机的一些寄存器。
我们可以通过 `Get` 和 `Put` 操作来读写这些寄存器。

客户机状态结构体为 `VexGuestXXXState`，位于 `VEX/pub/libvex_guest_XXX.h`。
其中 `XXX` 指代任意指架构名，如 `X86`、`MIPS64`。

### 运算符（operation）

此处的运算符类似计算机指令的操作数，包括各种算术运算、位运算等等。

运算符可以分为整数运算符、浮点运算符和向量运算符三类，共有上千种，还是非常多的。

运算符的定义如下：

```c
typedef enum {
    Iop_INVALID=0x1400,
    Iop_Add8,  Iop_Add16,  Iop_Add32,  Iop_Add64,
    Iop_Sub8,  Iop_Sub16,  Iop_Sub32,  Iop_Sub64,
    ...
} IROp;
```

### 表达式（expression）

表达式由运算符和运算对象组成，代表了一个被计算出来的值或者常量，本身没有副作用。

表达式的定义如下：

```c
typedef enum {
    Iex_Binder=0x1900,
    Iex_Get,
    Iex_GetI,
    Iex_RdTmp,
    Iex_Qop,
    Iex_Triop,
    Iex_Binop,
    Iex_Unop,
    Iex_Load,
    Iex_Const,
    Iex_ITE,
    Iex_CCall,
    Iex_VECRET,
    Iex_GSPTR
} IRExprTag;

typedef struct _IRExpr IRExpr;
struct _IRExpr {
    IRExprTag tag;
    union {
        struct {
            Int binder;
        } Binder;
        struct {
            Int    offset;
            IRType ty;
        } Get;
    } Iex;
};
```

### 临时变量（temporary variable）

在执行过程中，IR 表达式可能被存于临时变量，临时变量的类型（type）包括了整数、浮点数和向量三大类，具体细分如下：

```c
typedef enum {
    Ity_INVALID=0x1100,
    Ity_I1,
    Ity_I8,
    Ity_I16,
    Ity_I32,
    Ity_I64,
    Ity_I128,  /* 128-bit scalar */
    Ity_F16,   /* 16 bit float */
    Ity_F32,   /* IEEE 754 float */
    Ity_F64,   /* IEEE 754 double */
    Ity_D32,   /* 32-bit Decimal floating point */
    Ity_D64,   /* 64-bit Decimal floating point */
    Ity_D128,  /* 128-bit Decimal floating point */
    Ity_F128,  /* 128-bit floating point; implementation defined */
    Ity_V128,  /* 128-bit SIMD */
    Ity_V256   /* 256-bit SIMD */
} IRType;
```

### 语句（statement）

语句由表达式和变量组成，会修改客户机状态，是具有副作用的。

语句的定义如下：

```c
typedef enum {
    Ist_NoOp=0x1E00,
    Ist_IMark,
    Ist_AbiHint,
    Ist_Put,
    Ist_PutI,
    Ist_WrTmp,
    Ist_Store,
    Ist_LoadG,
    Ist_StoreG,
    Ist_CAS,
    Ist_LLSC,
    Ist_Dirty,
    Ist_MBE,
    Ist_Exit
} IRStmtTag;

typedef struct _IRStmt {
    IRStmtTag tag;
    union {
        struct {
        } NoOp;
        struct {
            Addr   addr;
            UInt   len;
            UChar  delta;
        } IMark;
        struct {
            IRExpr* base;
            Int     len;
            IRExpr* nia;
        } AbiHint;
        ...
    } Ist;
} IRStmt;
```

### 代码块（code blocks）

代码被拆分成了很多个代码块，也叫中间语言超级块（IR super blocks, IRSB）。
每个 IRSB 包含了 1 到 50 条指令。

IRSB 只有唯一的入口，但可以有多个出口（single-entry, multiple-exit）。
每个 IRSB 主要由以下三部分组成：

- 一个类型环境（type environment），用于表示该 IRSB 中每个临时变量的类型。
- 一个语句列表（list of statements），用于表示代码。
- 一个跳转（jump），用于表示如何从 IRSB 中退出。

IRSB 的定义如下：

```c
typedef enum {
    Ijk_INVALID=0x1A00,
    Ijk_Boring,         /* not interesting; just goto next */
    Ijk_Call,           /* guest is doing a call */
    Ijk_Ret,            /* guest is doing a return */
    Ijk_ClientReq,      /* do guest client req before continuing */
    Ijk_Yield,          /* client is yielding to thread scheduler */
    Ijk_EmWarn,         /* report emulation warning before continuing */
    Ijk_EmFail,         /* emulation critical (FATAL) error; give up */
    Ijk_NoDecode,       /* current instruction cannot be decoded */
    Ijk_MapFail,        /* Vex-provided address translation failed */
    Ijk_InvalICache,    /* Inval icache for range [CMSTART, +CMLEN) */
    Ijk_FlushDCache,    /* Flush dcache for range [CMSTART, +CMLEN) */
    Ijk_NoRedir,        /* Jump to un-redirected guest addr */
    Ijk_SigILL,         /* current instruction synths SIGILL */
    Ijk_SigTRAP,        /* current instruction synths SIGTRAP */
    Ijk_SigSEGV,        /* current instruction synths SIGSEGV */
    Ijk_SigBUS,         /* current instruction synths SIGBUS */
    Ijk_SigFPE,         /* current instruction synths generic SIGFPE */
    Ijk_SigFPE_IntDiv,  /* current instruction synths SIGFPE - IntDiv */
    Ijk_SigFPE_IntOvf,  /* current instruction synths SIGFPE - IntOvf */
    /* Unfortunately, various guest-dependent syscall kinds.  They
    all mean: do a syscall before continuing. */
    Ijk_Sys_syscall,    /* amd64/x86 'syscall', ppc 'sc', arm 'svc #0' */
    Ijk_Sys_int32,      /* amd64/x86 'int $0x20' */
    Ijk_Sys_int128,     /* amd64/x86 'int $0x80' */
    Ijk_Sys_int129,     /* amd64/x86 'int $0x81' */
    Ijk_Sys_int130,     /* amd64/x86 'int $0x82' */
    Ijk_Sys_int145,     /* amd64/x86 'int $0x91' */
    Ijk_Sys_int210,     /* amd64/x86 'int $0xD2' */
    Ijk_Sys_sysenter    /* x86 'sysenter'.  guest_EIP becomes
                           invalid at the point this happens. */
} IRJumpKind;

typedef struct {
    IRType* types;
    Int     types_size;
    Int     types_used;
} IRTypeEnv;

typedef struct {
    IRTypeEnv* tyenv;
    IRStmt**   stmts;
    Int        stmts_size;
    Int        stmts_used;
    IRExpr*    next;
    IRJumpKind jumpkind;
    Int        offsIP;
} IRSB;
```

## libVEX 的重要函数

libVEX 的重要函数及其作用基本都在 `VEX/pub/libvex.h` 中写明了（注释超级多，可能是个人习惯不同，我觉得这么多注释已经影响阅读了，还是单独放文档里更好）。

### 初始化

`LibVEX_Init()` 函数位于 `VEX/priv/main_main.c`。

该函数设置了一些 libVEX 的参数，然后进行了大量的检查工作，诸如各参数的大小是否在合理范围、各种类型的大小是否与预期一致等等。

### 翻译

`LibVEX_Translate()`、`LibVEX_FrontEnd()` 和 `libvex_BackEnd()` 函数都位于 `VEX/priv/main_main.c`：

`LibVEX_Translate()` 是总的翻译函数，会调用编译器的前端和后端：

```c
VexTranslateResult LibVEX_Translate ( /*MOD*/ VexTranslateArgs* vta )
{
    VexTranslateResult res = { 0 };
    VexRegisterUpdates pxControl = VexRegUpd_INVALID;

    IRSB* irsb = LibVEX_FrontEnd(vta, &res, &pxControl);
    libvex_BackEnd(vta, &res, irsb, pxControl);
    return res;
}
```

`LibVEX_FrontEnd()` 作为编译器前端，主要干了这些工作：

1. 初始化一些变量，根据架构设置对应的功能函数。
2. 调用 `bb_to_IR()` 函数翻译二进制代码到 Vex IR 超级块。
3. 调用 `do_iropt_BB()` 函数进行优化（iropt: IR optimiser）。
4. 装上分析工具（get the thing instrumented），似乎主要是给 gdbserver 用的。
5. 进行一些后续的清理工作。

若设置了调试参数，会打印额外信息。同时，期间进行了很多理智检查，我略去了这些步骤，后文也是如此。

`libvex_BackEnd()` 作为编译器后端，主要干了这些工作：

1. 初始化一些变量。
2. 根据架构设置对应的功能函数。
3. 调用 `ado_treebuild_BB()` 函数建立树。
4. 调用 `iselSB_XXX()` 函数翻译 Vex IR 到架构相关的代码。
5. 调用分配器为虚拟寄存器分配实际的寄存器（大概率走 `doRegisterAllocation_v3()` 函数）。
6. 调用 `emit_XXX()` 函数发射二进制代码（汇编，assembly）。

## libVEX 在 Valgrind 中的使用

接着 [Valgrind 简介](/zh-cn/posts/valgrind/valgrind)，继续分析相关源码。

### 翻译（translate）

Valgrind 的翻译模块的代码位于 `coregrind/pub_core_translate.h` 和 `coregrind/m_translate.c`。

该模块是 libVEX 的即时编译器（JITter）的接口。

`VG_(translate)()` 主要干了以下的工作：

1. 若 libVEX 未初始化，调用 `LibVEX_Init()` 函数初始化。
2. 确立翻译类型和实际开始的客户机地址。
3. 输出重定向信息。
4. 若设置了调试参数，跟踪、打印一些额外信息。
5. 查找合适的内存段（读写、可执行）。
6. 根据调试参数，设置一些变量（`verbosity` 变量）。
7. 根据翻译类型设置序言函数。
8. 获取架构信息。
9. 设置 libVEX 的各种参数和分派相关的信息。
10. 调用 `LibVEX_Translate()` 函数实际执行翻译。
11. 进行一些统计工作。
12. 若设置了调试参数则打印一些额外信息。

### 调度器（续）

Valgrind 简介的[调度器（scheduler）](/zh-cn/posts/valgrind/valgrind#调度器scheduler)一节中提到了`run_thread_for_a_while()` 函数，现在我们具体分析一下该函数的主要流程：

1. 获取线程状态。
2. 初始化（清零）返回值 `two_words` 数组。
3. 设置主机代码地址：若主机代码没有重定向，主机代码地址就是传入的地址；若被重定向了（大多数情况），调用 `VG_(search_transtab)()` 函数查找然后设置为重定向后的地址。
4. 建立事件计数器。
5. 调用 `VG_(disp_run_translations)` 实际开始执行。
6. 处理客户端程序的 fault（若有的话）。
7. 处理事件相关的后续工作。
8. 处理 vgdb 相关的后续工作。
9. 检查返回值的合理性。

对于首次运行的客户端程序，在第 3 步中 `VG_(search_transtab)()` 会返回未找到，这时候 `run_thread_for_a_while()` 直接返回 `VG_TRC_INNER_FASTMISS`。

代码块的退出可能由多种情况造成，这时候需要根据退出状态来进行相应的处理。
比如，上文的 `VG_TRC_INNER_FASTMISS` 是因为 Valgrind 自身的原因退出（没有提前翻译），这部分退出状态的代码位于 `coregrind/pub_core_dispatch_asm.h`。
而很多时候是 libVEX 要求进行额外处理所以退出，比如 `VEX_TRC_JMP_SYS_SYSCALL` 表示要执行系统调用然后才能继续，这部分代码位于 `VEX/pub/libvex_trc_values.h`。

`VG_(scheduler)()` 的对不同返回值的处理大致如下：

- `VEX_TRC_JMP_NOREDIR`：调用 `handle_noredir_jump()` 函数。
- `VEX_TRC_JMP_BORING` 和 `VG_TRC_BORING`：啥也不做。
- `VG_TRC_INNER_FASTMISS`：对于缓存未命中的，调用 `handle_tt_miss()` 函数，该函数会调用 `VG_(translate)()` 函数进行[翻译](#翻译)工作。
- 系统调用相关的：调用 `handle_syscall()` 函数执行系统调用。
- 信号相关的：调用相应的信号处理函数。
- `VG_TRC_CHAIN_ME_TO_SLOW_EP` 和 `VG_TRC_CHAIN_ME_TO_FAST_EP`：调用 `handle_chain_me()` 函数打补丁。
- ……

## 移植 libVEX

有时候我们需要为 libVEX 添加更多的指令甚至架构支持，我学习 libVEX 的最终目的是为了适配 LoongArch，这部分内容见 [Valgrind for LoongArch](/zh-cn/posts/valgrind/loongarch)。
