---
title: "Valgrind for LoongArch"
date: 2021-12-27
description: ""
menu:
  sidebar:
    name: "Valgrind for LoongArch"
    identifier: valgrind-loongarch
    parent: valgrind
    weight: 100
tags: ["Valgrind", "LoongArch"]
categories: ["Valgrind"]
---

简单介绍移植 Valgrind for LoongArch 的过程。

<!--more-->

## 引言

好久没更新了，目前基本完成了 Valgrind for LoongArch 的适配，补丁的整理参考了 [riscv64](https://github.com/petrpavlu/valgrind-riscv64)。

项目仓库见 <https://github.com/FreeFlyingSheep/valgrind-loongarch64>，通过了除 `drd/tests/pth_mutex_signal` 以外的所有回归测试，且这个未通过项目是已知的公共问题。

向上游邮件列表发送了[邮件](https://sourceforge.net/p/valgrind/mailman/valgrind-developers/thread/CACWXhK%3DjZ8_Wpu8meJOjYFxX5AgbQ3Ad7BdFW19vRWoRhT-fmA%40mail.gmail.com/#msg37651955)，进一步跟进中，希望能早日合入上游。

## LibVEX

我实际完成 LibVEX 的移植是在 Valgrind 主体的移植后的，当时考虑的是先在本地通过编译跑起来再说。
但理论上应该先完成 LibVEX 的移植。

### 前端

前端主要包括三个文件：`VEX/priv/guest_loongarch64_defs.h`、`VEX/priv/guest_loongarch64_helpers.c` 和 `VEX/priv/guest_loongarch64_toIR.c`。

前端的设计主要参考了 arm64 和 mips64，不过我把所有指令的解析都分拆在各个函数里。

指令的分类我参考了 QEMU 的 LoongArch 补丁，翻译的内容见[汇总](#汇总)，解码部分每几位归为一组，目的是尽可能凑 `switch` 函数的跳转表。

对于不方便翻译为 Vex IR 的指令，采用函数调用的方式直接交给帮助函数处理，比如一些 `crc` 指令等。

### 后端

后端主要包括三个文件：`VEX/priv/host_loongarch64_defs.h`、`VEX/priv/host_loongarch64_isel.c` 和 `VEX/priv/host_loongarch64_defs.c`。

`host_loongarch64_defs.h` 包括了各种指令的标签（用枚举定义）和结构体定义，方便起见，我直接把指令标签的枚举值定义为指令的操作码。

`host_loongarch64_isel.c` 负责把所有 Vex IR 转换成 LoongArch 指令的结构体，具体内容见[汇总](#汇总)。

`host_loongarch64_defs.c` 负责把 LoongArch 指令的结构体转换成二进制，发射出去。

其中后端设计的原子操作（主要是 CAS）我直接抄了内核的实现（`arch/loongarch/kernel/cmpxchg.c`）。

### 汇总

用到的字节序：

- `Iend_LE`：LoongArch 只有小端序

用到的常量：

- `Ico_F32i`：主要用于表示浮点数 `1.0`
- `Ico_F64i`：主要用于表示浮点数 `1.0`
- `Ico_U16`：主要用于表示 16 位立即数
- `Ico_U32`：主要用于表示 32 位立即数
- `Ico_U64`：主要用于表示 `pc` 寄存器和 64 位立即数
- `Ico_U8`：主要用于表示 `ui5`、`ui6`、`sa2`、`sa3`、`msbw`、`lsbw`、`msbd`、`lsbd` 和 8 位立即数
- `Ico_U1`：主要用于表示比较运算的结果

用到的类型：

- `Ity_F32`：主要用于浮点寄存器的低 32 位
- `Ity_F64`：主要用于浮点寄存器
- `Ity_I16`：主要用于整数寄存器的低 16 位
- `Ity_I32`：主要用于整数寄存器的低 32 位
- `Ity_I64`：主要用于整数寄存器
- `Ity_I8`：主要用于整数寄存器的低 8 位和 `fcc` 寄存器
- `Ity_I1`：主要用于整数寄存器的低 1 位

用到的操作符（浮点指令基本都涉及 `fcsr` 的保存和恢复）：

- `Iop_128HIto64`：`return hi`
- `Iop_128to64`：`return lo`
- `Iop_16Sto64`：`ext.w.h dst, src`
- `Iop_16Uto64`：`slli.d dst, src, 48; srli.d dst, dst, 48`
- `Iop_1Sto64`：`slli.d dst, src, 63; srai.d dst, dst, 63`
- `Iop_1Uto64`：`andi dst, src, 0x1`
- `Iop_1Uto8`：`andi dst, src, 0x1`
- `Iop_32Sto64`：`slli.w dst, src, 0`
- `Iop_32Uto64`：`slli.d dst, src, 32; srli.d dst, dst, 32`
- `Iop_32to8`：`andi dst, src, 0xff`
- `Iop_64HIto32`：`srli.d dst, src, 32`
- `Iop_64to32`：`slli.d dst, src, 32; srli.d dst, dst, 32`
- `Iop_64to8`：`andi dst, src, 0xff`
- `Iop_8Sto64`：`ext.w.b dst, src`
- `Iop_8Uto32`：`andi dst, src, 0xff`
- `Iop_8Uto64`：`andi dst, src, 0xff`
- `Iop_AbsF32`：`fabs.s dst, src`
- `Iop_AbsF64`：`fabs.d dst, src`
- `Iop_Add32`：`add[i].w dst, src1, src2`
- `Iop_Add64`：`add[i].d dst, src1, src2`
- `Iop_AddF32`：`fadd.s dst, src1, src2`
- `Iop_AddF64`：`fadd.d dst, src1, src2`
- `Iop_And32`：`and[i] dst, src1, src2`
- `Iop_And64`：`and[i] dst, src1, src2`
- `Iop_CasCmpNE32`：`xor dst, src1, src2; sltu dst, $zero, dst`
- `Iop_CasCmpNE64`：`xor dst, src1, src2; sltu dst, $zero, dst`
- `Iop_Clz32`：`clz.w dst, src`
- `Iop_Clz64`：`clz.d dst, src`
- `Iop_CmpEQ32`：`xor dst, src1, src2; sltui dst, dst, 1`
- `Iop_CmpEQ64`：`xor dst, src1, src2; sltui dst, dst, 1`
- `Iop_CmpF32`：`fcmp.cond.s dst, src1, src2`
- `Iop_CmpF64`：`fcmp.cond.d dst, src1, src2`
- `Iop_CmpLT32S`：`slli.w src1, src1, 0; slli.w src2, src2, 0; slt dst, src1, src2`
- `Iop_CmpLT32U`：`slli.w src1, src1, 0; slli.w src2, src2, 0; sltu dst, src1, src2`
- `Iop_CmpLE64S`：`slt dst, src2, src1; nor dst, src, $zero`
- `Iop_CmpLE64U`：`sltu dst, src2, src1; nor dst, src, $zero`
- `Iop_CmpLT64S`：`slt dst, src1, src2`
- `Iop_CmpLT64U`：`sltu dst, src1, src2`
- `Iop_CmpNE32`：`xor dst, src1, src2; sltu dst, $zero, dst`
- `Iop_CmpNE64`：`xor dst, src1, src2; sltu dst, $zero, dst`
- `Iop_Ctz32`：`ctz.w dst, src`
- `Iop_Ctz64`：`ctz.d dst, src`
- `Iop_DivF32`：`fdiv.s dst, src1, src2`
- `Iop_DivF64`：`fdiv.d dst, src1, src2`
- `Iop_DivModS32to32`：`div.w lo, src1, src2; mod.w hi, src1, src2; slli.d hi, hi, 6; or dst, lo, hi`
- `Iop_DivModS64to64`：`div.d lo, src1, src2; mod.d hi, src1, src2`
- `Iop_DivModU32to32`：`div.wu lo, src1, src2; mod.wu hi, src1, src2; slli.d hi, hi, 6; or dst, lo, hi`
- `Iop_DivModU64to64`：`div.du lo, src1, src2; mod.du hi, src1, src2`
- `Iop_DivS32`：`div.w dst, src1, src2`
- `Iop_DivS64`：`div.wu dst, src1, src2`
- `Iop_DivU32`：`div.d dst, src1, src2`
- `Iop_DivU64`：`div.du dst, src1, src2`
- `Iop_F32toF64`：`fcvt.s.d dst, src`
- `Iop_F32toI32S`：`ftint.w.s dst, src`
- `Iop_F32toI64S`：`ftint.l.s dst, src`
- `Iop_F64toF32`：`fcvt.s.d dst, src`
- `Iop_F64toI32S`：`ftint.w.d dst, src`
- `Iop_F64toI64S`：`ftint.l.d dst, src`
- `Iop_I32StoF32`：`ffint.s.w dst, src`
- `Iop_I32StoF64`：`ffint.d.w dst, src`
- `Iop_I64StoF32`：`ffint.s.l dst, src`
- `Iop_I64StoF64`：`ffint.d.l dst, src`
- `Iop_MAddF32`：`fmadd.s dst, src1, src2, src3`
- `Iop_MAddF64`：`fmadd.d dst, src1, src2, src3`
- `Iop_MSubF32`：`fmsub.s dst, src1, src2, src3`
- `Iop_MSubF64`：`fmsub.d dst, src1, src2, src3`
- `Iop_MaxNumAbsF32`：`fmaxa.s dst, src1, src2`
- `Iop_MaxNumAbsF64`：`fmaxa.d dst, src1, src2`
- `Iop_MinNumAbsF32`：`fmina.s dst, src1, src2`
- `Iop_MinNumAbsF64`：`fmina.d dst, src1, src2`
- `Iop_MaxNumF32`：`fmax.s dst, src1, src2`
- `Iop_MaxNumF64`：`fmax.d dst, src1, src2`
- `Iop_MinNumF32`：`fmin.s dst, src1, src2`
- `Iop_MinNumF64`：`fmin.d dst, src1, src2`
- `Iop_MulF32`：`fmul.s dst, src1, src2`
- `Iop_MulF64`：`fmul.d dst, src1, src2`
- `Iop_MullS32`：`mulw.d.w dst, src1, src2`
- `Iop_MullS64`：`mul.d lo, src1, src2; mulh.d hi, src1, src2`
- `Iop_MullU32`：`mulw.d.wu dst, src1, src2`
- `Iop_MullU64`：`mul.d lo, src1, src2; mulh.du hi, src1, src2`
- `Iop_NegF32`：`fneg.s dst, src`
- `Iop_NegF64`：`fneg.d dst, src`
- `Iop_Not32`：`nor dst, src, $zero`
- `Iop_Not64`：`nor dst, src, $zero`
- `Iop_Or1`：`or dst, src1, src2`
- `Iop_Or32`：`or[i] dst, src1, src2`
- `Iop_Or64`：`or[i] dst, src1, src2`
- `Iop_RSqrtF32`：`frsqrt.s dst, src`
- `Iop_RSqrtF64`：`frsqrt.d dst, src`
- `Iop_ReinterpF32asI32`：`movfr2gr.s dst, src`
- `Iop_ReinterpF64asI64`：`movfr2gr.d dst, src`
- `Iop_ReinterpI32asF32`：`movgr2fr.w dst, src`
- `Iop_ReinterpI64asF64`：`movgr2fr.d dst, src`
- `Iop_RoundF32toInt`：`frint.s dst, src`
- `Iop_RoundF64toInt`：`frint.d dst, src`
- `Iop_Sar32`：`sra[i].w dst, src1, src2`
- `Iop_Sar64`：`sra[i].d dst, src1, src2`
- `Iop_Shl32`：`sll[i].w dst, src1, src2`
- `Iop_Shl64`：`sll[i].d dst, src1, src2`
- `Iop_Shr32`：`srl[i].w dst, src1, src2`
- `Iop_Shr64`：`srl[i].d dst, src1, src2`
- `Iop_SqrtF32`：`fsqrt.s dst, src`
- `Iop_SqrtF64`：`fsqrt.s dst, src`
- `Iop_Sub32`：`sub.w dst, src1, src2`
- `Iop_Sub64`：`sub.d dst, src1, src2`
- `Iop_SubF32`：`fsub.s dst, src1, src2`
- `Iop_SubF64`：`fsub.d dst, src1, src2`
- `Iop_Xor32`：`xor[i] dst, src1, src2`
- `Iop_Xor64`：`xor[i] dst, src1, src2`

用到的表达式：

- `Iex_Binop`：用于表示二元操作符和对应的操作数
- `Iex_Const`：用于表示常量
- `Iex_Get`：用于读寄存器
- `Iex_ITE`：用于表示 `if-else` 操作
- `Iex_Load`：用于读内存
- `Iex_Qop`：用于表示四元操作符和对应的操作数
- `Iex_RdTmp`：用于表示读取临时变量
- `Iex_Triop`：用于表示三元操作符和对应的操作数
- `Iex_Unop`：用于表示一元操作符和对应的操作数

用到的跳转方式：

- `Ijk_Boring`：用于表示普通的跳转
- `Ijk_ClientReq`：用于表示需要处理客户端请求（Valgrind 专用）
- `Ijk_SigFPE_IntDiv`：用于表示当前指令触发 `SIGFPE` 异常，且是整数除 0 异常
- `Ijk_SigFPE_IntOvf`：用户表示当前指令触发 `SIGFPE` 异常，且是整数溢出异常
- `Ijk_INVALID`：用于表示无效（libVEX 本身出现错误）
- `Ijk_InvalICache`：用于表示需要使指令缓存无效（Valgrind 专用）
- `Ijk_NoDecode`：用于表示解码失败
- `Ijk_NoRedir`：用于表示跳转到未重定向到地址（Valgrind 专用）
- `Ijk_SigBUS`：用于表示当前指令触发 `SIGBUS` 异常
- `Ijk_SigILL`：用于表示当前指令触发 `SIGILL` 异常
- `Ijk_SigSEGV`：用于表示当前指令触发 `SIGSEGV` 异常
- `Ijk_SigSYS`：用于表示当前指令触发 `SIGSYS` 异常
- `Ijk_SigTRAP`：用于表示当前指令触发 `SIGTRAP` 异常
- `Ijk_Sys_syscall`：用于表示需要进行系统调用

用到的语句：

- `Ist_CAS`：用于原子操作（Compare And Swap）
- `Ist_Exit`：用于表示退出
- `Ist_LLSC`：用于原子操作（`ll`/`sc`）
- `Ist_MBE`：用于表示内存屏障
- `Ist_Put`：用于表示写寄存器
- `Ist_Store`：用于表示写内存
- `Ist_WrTmp`：用于表示给临时变量赋值

Valgrind 特殊指令：

```text
guard:
srli.d $zero, $zero, 3
srli.d $zero, $zero, 13
srli.d $zero, $zero, 29
srli.d $zero, $zero, 19

$a7 = client_request($t0):
or $t1, $t1, $t1

$a7 = guest_NRADDR:
or $t2, $t2, $t2

call-noredir $t8:
or $t3, $t3, $t3

IR injection:
or $t4, $t4, $t4
```

公共框架添加的内容：

- `Ist_MBE`：添加表示指令屏障的功能
- `Iop_LogBF32`：`flogb.s dst, src`
- `Iop_LogBF64`：`flogb.d dst, src`
- `Iop_ScaleBF32`：`fscaleb.s dst, src1, src2`
- `Iop_ScaleBF64`：`fscaleb.s dst, src1, src2`

除此以外，还有其他 Valgrind 工具（如 memcheck）会用到一些专有操作符（如 `Iop_CmpNEZ8`），也需要在后端翻译，此处略去。

## Valgrind

Valgrind 主体的移植第一步是解决编译问题，几乎所有需要修改的地方都有 `#error` 提示，顺着编译报错加入 LoongArch 的 `#ifdef`，不确定的地方先写 `/* TODO */`。

在顺利通过了编译后，再开始分模块完善代码，下面介绍几个关键模块的移植思路。

### gdbserver

gdbserver 部分的代码主要位于 `coregrind/m_gdbserver/valgrind-low-loongarch64.c`。

这部分代码参考了 GDB 的实现（`gdb/gdbserver/linux-loongarch-low.c`），因为新版本 GDB 的相关代码已经重构过，所以要适当调整（不能照抄，有点难受）。

测试的时候使用 `--vgdb=yes --vgdb-error=0` 参数。

### sigframe

信号栈部分的代码主要位于 `coregrind/m_sigframe/sigframe-loongarch64-linux.c`。

这部分代码参考内核信号栈的实现（`arch/loongarch/kernel/signal.c`）。

目前只考虑整数寄存器，暂不实现浮点寄存器和二进制翻译寄存器、向量寄存器的存取。

### dispatch

分派部分的代码主要位于 `coregrind/m_dispatch/dispatch-loongarch64-linux.S`。

这部分代码参考 mips64 的实现，最大的区别是要加上指令屏障。

### cache

缓存部分的代码主要位于 `coregrind/m_cache.c`。

这部分代码参考内核 CPU 探测部分（`arch/loongarch/mm/cache.c`），利用 `cpucfg` 指令实现了用户态下对 CPU 缓存属性的读取。

### machine

机器探测部分的代码主要位于 `coregrind/m_machine.c`。

似乎可以借助 `cpucfg` 指令，但这样有一个问题，硬件支持但内核不支持的属性，应用软件同样无法使用。

所以参考 mips 读取 `/proc/cpuinfo` 来完成相应设置。

## 踩坑

### 指令屏障

执行同一个程序，有时候会 SIGILL，大部分时候又能正常运行，用 GDB 追踪时却总是好的。

索性在内核添加打印，发现触发 SIGILL 的指令编码永远是 `0`。

后来想了想，可能是需要添加指令屏障，在 dispatch 模块跳转到生成的指令前加了 `ibar` 指令，之后果然好了。

### 32 位除法结果不对

测试发现 32 位除法指令的结果是不对的，但我怎么都觉得自己翻译代码没写错，一脸懵逼。

手册上在除法指令介绍完以后只写了一句话，描述如果超过 32 位，则结果为无意义的任何值。

我一开始理解为就是高 32 位必须为 `0`，但并不是这个意思。
单独验证除法指令，实际 CPU 执行时，对于正数，高 32 位必须为 `0`，而对于负数，必须高位全 `1`。
因为第 32 位为 `1` 且高 32 位为 `0` 被解释为正数，而这个数超了 32 位整数的范围。

一个简单粗暴的解决方案是把 32 位除法指令的操作数统一进行符号扩展。

### 地址解析问题

经过不停的修改测试，好不容易把静态链接的小程序跑通了，但一运行动态链接程序就段错误，让人费解。

由于是挂在动态链接器里，所以比较难追踪，为此我甚至写了不少脚本去测试单指令的正确性，以及比较不同日志里面涉及的指令，见[脚本](#脚本)。

某天突然发现了端倪，有大量 `ld` 指令的立即数是很大的负数，这种现象不正常。

再次检查 `VEX/priv/host_loongarch64_isel.c` 中的代码，`ld` 指令立即数的最高位是符号位，因此解析地址时（`iselIntExpr_AMode_wrk()`），超过 11 位的立即数就应该走 `ldx` 指令，而我判断的时候用的是 `0xfff`，改成 `0x7ff` 就解决了问题。

### 来自 C 库的警告

使用 memcheck 跑动态链接程序的时候，发现一万个警告，而在 x86 下运行是没这些警告的。

后来发现可以通过 `*.supp` 文件忽略 C 库的警告，于是我暂时偷懒直接在 `glibc-2.X.supp.in` 中把 `*/libc.so*` 和 `*/ld-linux-loongarch-*.so*` 文件产生的警告给忽略了。

```text
##----------------------------------------------------------------------##
# LoongArch64 Linux
{
   glibc-loongarch64-cond-1
   Memcheck:Cond
   obj:*/libc.so*
}
{
   glibc-loongarch64-cond-2
   Memcheck:Cond
   obj:*/ld-linux-loongarch-*.so*
}
```

在完成了一些验证后，我发现报错的来源主要是 `sc.w` 指令，我怀疑是我模拟的有问题。

在逐步排查所有 `exit` 的地方后，发现问题大概是出自 `guest_LLSC_DATA` 和 `data` 的比较上，因为 `guest_LLSC_DATA` 是一个 64 位的变量，我把 32 位的 `data` 也转成 64 位后， `sc` 指令产生的报错就消失了。

其他报错来自于 C 位域，反汇编后发现和 `bstrins` 指令有关，推测是我偷懒用 C 语言函数模拟，导致 memcheck 不能正确跟踪位的情况，于是老老实实用多条 Vex IR 指令模拟，之后发现 memcheck 关于 C 库的报错全没了，太爽了。

除了 memcheck，helgrind 也有来自 C 库的警告，最终发现是需要在 `coregrind/m_redir.c` 中添加动态链接库的名字。

### 结构体要及时同步

运行 `valgrind --tool=none gcc --version` 结果段错误了，用 GDB 追踪发现系统调用传递的参数都错位了。

排查了半天发现是新版本内核结构体变动了，没及时跟上（内核还没进社区，每一版改动可能较大），这蛋疼的 Valgrind 内核接口模块，每次都要手动去更新。

### GDB

每次测试 Valgrind 的 gdbserver 功能都提示 `Remote 'g' packet reply is too long`。

查看社区版 GDB 源码后发现尽管写了代码，但竟然默认不支持浮点寄存器（`gdb/arch/loongarch.c`），我把 `coregrind/m_gdbserver/valgrind-low-loongarch64.c` 中浮点寄存器代码注释掉后就好了。

最后改用了内部完整版的 GDB 进行测试，把浮点寄存器加回去了。

### VDSO

每次在中文环境下执行 `valgrind --tool=none ls -l` 就会段错误，一直以为是指令翻译有问题，追了好久发现是信号处理的时候出了问题。

中文环境下 `valgrind ls -l` 默认客户栈不够，此时内核发送 `SIGSEGV`，Valgrind 信号处理函数会调用 `mmap()` 系统调用扩展栈，之后返回的时候到了 VDSO 的 `sigreturn()` 函数。

这时候问题就来了，Valgrind 的地址空间管理器默认会移除 VDSO 的映射，这时候跳转到的地址是非法的，因此再次触发 `SIGSEGV`，程序退出。

而高版本内核不支持 `SA_RESTORER`，因此没法向其他架构一样用自己的 `my_sigreturn()` 函数。

后来追踪历史记录发现在 `coregrind/m_initimg/initimg-linux.c` 中不取消映射就行了（添加 `#ifdef`）。

### LLSC

LoongArch 下的 `ll`/`sc` 指令对，中间不能有其他访存指令，不然一定会失败。

而要保证 Valgrind 不在 `ll`/`sc` 指令对中使用其他访存指令是非常难的，目前并没有架构实现。

这个问题导致了用到 LLSC 的程序几乎都死循环了，于是我实现了 fallback 版本的 LLSC（即用软件模拟），并强制启用它。

### `AM*` 原子指令

有时候遇到 `AM*` 原子指令的程序会死循环，而稍微改改前端代码，增加或者删掉几个临时 Vex IR 变量又突然好了，非常迷。

在 GDB 里单指令跟踪，发现最后发射的 `ll`/`sc` 指令对总是失败，查看寄存器内容是 `ll.w` 符号扩展了，而比较的时候，另一个操作数还是 32 位的，必定失败。

对另一个操作数进行符号扩展，总算解决了这个问题。

### 接收 `SIGINT` 后直接报错

在部分多线程程序中按 `ctrl + c`，会导致 `coregrind/m_signals.c` 中 `async_signalhandler()` 函数的 `vg_assert(tst->status == VgTs_WaitSys);` 失败。

搜索历史记录，有不少架构也修过这个问题，所以一开始怀疑是哪里需要加东西，但怎么都找不到。
后来追踪 Valgrind 大锁和信号的 mask，发现该阻塞的地方没阻塞，这一定是哪里设置出了问题。

阻塞信号用的是 `rt_sigprocmask()` 系统调用，查找用到这个系统调用的地方，发现 `coregrind/m_syswrap/syscall-loongarch64-linux.S` 最可疑。
比照 arm64，果然抄错了一个地方，第二次重新阻塞时应该用 `postmask` 而不是 `syscall_mask`。
修改后程序能正常处理 `SIGINT`，总算告一段落。

### 加载立即数

为了优化大量的加载立即数指令，根据立即数大小分为三种情况：

```c
if ((imm >> 12) == 0)
    p = mkLoadImm_EXACTLY1(p, dst, imm);
else if (imm < 0xffffffff || (imm >> 31) == 0x1ffffffffUL)
    p = mkLoadImm_EXACTLY2(p, dst, imm);
else
    p = mkLoadImm_EXACTLY4(p, dst, imm);
return p;
```

使用一条指令加载：

```text
ori dst, $zero, imm[11:0]
```

使用两条指令加载：

```text
lu12i.w dst, imm[31:12]
ori     dst, dst, imm[11:0]
```

使用四条指令加载：

```text
lu12i.w dst, imm[31:12]
ori     dst, dst, imm[11:0]
lu32i.d dst, imm[51:32]
lu52i.d dst, dst, imm[63:52]
```

在验证 `pcaddu12i` 和 `pcalau12i` 指令时发现问题，高位应该为 `0` 的情况变成全 `1` 了。
追踪发现是加载立即数是，希望加载一个第 32 位为 `1` 的 64 位立即数，但被误判为了使用两条指令去加载 32 位立即数。
解决办法是修改 `imm < 0xffffffff` 为 `imm < 0x80000000`，确保这种情况下使用四条指令去加载。

### `movcf2gr` 和 `movcf2fr`

`movcf2gr` 和 `movcf2fr` 指令会清空高位，手册没写明。

### `clone3()` 系统调用

Valgrind 并未实现 `clone3()` 系统调用相关的代码，因此我参考其他架构，直接在 `coregrind/m_syswrap/syswrap-loongarch64-linux.c` 的系统调用表里，把 `clone3()` 设置为 `sys_ni_syscall`。

## 脚本

记录一些方便自己测试、调试的小脚本。本着能用就行的原则，写得很烂（其实是太菜，逃

### 验证单指令正确性

为了绕开 glibc，自己实现打印、退出函数（通过系统调用），程序开始执行时保存所有寄存器，这部分用汇编实现：

```text
.text
.globl _start
_start:
    nop

dump:
    la.local   $x,   regs
    st.d       $r0,  $x,  0
    st.d       $r1,  $x,  8
    st.d       $r2,  $x,  16
    st.d       $r3,  $x,  24
    st.d       $r4,  $x,  32
    st.d       $r5,  $x,  40
    st.d       $r6,  $x,  48
    st.d       $r7,  $x,  56
    st.d       $r8,  $x,  64
    st.d       $r9,  $x,  72
    st.d       $r10, $x,  80
    st.d       $r11, $x,  88
    st.d       $r12, $x,  96
    st.d       $r13, $x,  104
    st.d       $r14, $x,  112
    st.d       $r15, $x,  120
    st.d       $r16, $x,  128
    st.d       $r17, $x,  136
    st.d       $r18, $x,  144
    st.d       $r19, $x,  152
    st.d       $r20, $x,  160
    st.d       $r21, $x,  168
    st.d       $r22, $x,  176
    st.d       $r23, $x,  184
    st.d       $r24, $x,  192
    st.d       $r25, $x,  200
    st.d       $r26, $x,  208
    st.d       $r27, $x,  216
    st.d       $r28, $x,  224
    st.d       $r29, $x,  232
    st.d       $r30, $x,  240
    st.d       $r31, $x,  248
    fst.d      $f0,  $x,  256
    fst.d      $f1,  $x,  264
    fst.d      $f2,  $x,  272
    fst.d      $f3,  $x,  280
    fst.d      $f4,  $x,  288
    fst.d      $f5,  $x,  296
    fst.d      $f6,  $x,  304
    fst.d      $f7,  $x,  312
    fst.d      $f8,  $x,  320
    fst.d      $f9,  $x,  328
    fst.d      $f10, $x,  336
    fst.d      $f11, $x,  344
    fst.d      $f12, $x,  352
    fst.d      $f13, $x,  360
    fst.d      $f14, $x,  368
    fst.d      $f15, $x,  376
    fst.d      $f16, $x,  384
    fst.d      $f17, $x,  392
    fst.d      $f18, $x,  400
    fst.d      $f19, $x,  408
    fst.d      $f20, $x,  416
    fst.d      $f21, $x,  424
    fst.d      $f22, $x,  432
    fst.d      $f23, $x,  440
    fst.d      $f24, $x,  448
    fst.d      $f25, $x,  456
    fst.d      $f26, $x,  464
    fst.d      $f27, $x,  472
    fst.d      $f28, $x,  480
    fst.d      $f29, $x,  488
    fst.d      $f30, $x,  496
    fst.d      $f31, $x,  504
    movfcsr2gr $t0,  $r0
    movfcsr2gr $t1,  $r1
    movfcsr2gr $t2,  $r2
    movfcsr2gr $t3,  $r3
    st.d       $t0,  $x,  512
    st.d       $t1,  $x,  520
    st.d       $t2,  $x,  528
    st.d       $t3,  $x,  536
    movcf2gr   $t0,  $fcc0
    andi       $t0,  $t0, 1
    movcf2gr   $t1,  $fcc1
    andi       $t1,  $t1, 1
    movcf2gr   $t2,  $fcc2
    andi       $t2,  $t2, 1
    movcf2gr   $t3,  $fcc3
    andi       $t3,  $t3, 1
    movcf2gr   $t4,  $fcc4
    andi       $t4,  $t4, 1
    movcf2gr   $t5,  $fcc5
    andi       $t5,  $t5, 1
    movcf2gr   $t6,  $fcc6
    andi       $t6,  $t6, 1
    movcf2gr   $t7,  $fcc7
    andi       $t7,  $t7, 1
    st.d       $t0,  $x,  544
    st.d       $t1,  $x,  552
    st.d       $t2,  $x,  560
    st.d       $t3,  $x,  568
    st.d       $t4,  $x,  576
    st.d       $t5,  $x,  584
    st.d       $t6,  $x,  592
    st.d       $t7,  $x,  600
    move       $a0,  $x
    bl         show

exit:
    li.w    $a7, 93   // n = __NR_exit
    syscall 0         // exit(ret)

.globl print
print:
    move    $a2, $a1  // l
    move    $a1, $a0  // s
    li.w    $a0, 2    // f = stderr
    li.w    $a7, 64   // n = __NR_write
    syscall 0         // write(f, s, l)
    jr      $ra
```

之后跳转到 C 函数去打印保存的寄存器：

```c
void print(char *s, int n);

unsigned long regs[32 + 32 + 4 + 8];

int my_strlen(char *s)
{
    char *t = s;
    while (*s != '\0')
        s++;
    return s - t;
}

void my_puts(char *s)
{
    print(s, my_strlen(s));
}

void my_itoa(char *s, unsigned long n, int base)
{
    char t[20];
    char a[] = { '0', '1', '2', '3', '4', '5', '6', '7',
                 '8', '9', 'a', 'b', 'c', 'd', 'e', 'f' };
    int i = 0;
    if (base == 16) {
        *s++ = '0';
        *s++ = 'x';
    } else if ((long) n < 0) {
        *s++ = '-';
        n = -n;
    }
    do {
        t[i++] = a[n % base];
        n /= base;
    } while (n != 0);
    while (--i >= 0)
        *s++ = t[i];
    *s = '\0';
}

void widen(char *dst, char *src, int len)
{
    *dst++ = *src++; // '0'
    *dst++ = *src++; // 'x'
    for (int i = my_strlen(src); i < len; i++)
        *dst++ = '0';
    while ((*dst++ = *src++) != '\0')
        continue;
}

int show(unsigned long *regs)
{
    char s[20], t[20];
    for (int i = 0; i < 32; i++) {
        my_puts("r");
        my_itoa(s, i, 10);
        my_puts(s);
        my_puts(":\t");
        my_itoa(s, regs[i], 16);
        widen(t, s, 16);
        my_puts(t);
        my_puts("\n");
    }
    for (int i = 0; i < 32; i++) {
        my_puts("f");
        my_itoa(s, i, 10);
        my_puts(s);
        my_puts(":\t");
        my_itoa(s, regs[i + 32], 16);
        widen(t, s, 16);
        my_puts(t);
        my_puts("\n");
    }
    for (int i = 0; i < 4; i++) {
        my_puts("fcsr");
        my_itoa(s, i, 10);
        my_puts(s);
        my_puts(":\t");
        my_itoa(s, regs[i + 64], 16);
        widen(t, s, 8);
        my_puts(t);
        my_puts("\n");
    }
    for (int i = 0; i < 8; i++) {
        my_puts("fcc");
        my_itoa(s, i, 10);
        my_puts(s);
        my_puts(":\t");
        my_itoa(s, regs[i + 68], 16);
        my_puts(s);
        my_puts("\n");
    }
    return 0;
}
```

只需要把 `nop` 替换为需要验证的非转移指令，上述代码就可以打印运行指令后的寄存器状态。
于是之后借助一个 python 脚本来实现不同指令的填充和验证工作：

```python
import os
import numpy as np

def write(line, file):
    with open("dump.S", "r") as input:
        text = input.read()
        with open(file, "w") as output:
            output.write(text.replace("nop\n", line))

def build(line):
    write(line, "tmp.S")
    ret = os.system("gcc -nostdlib -static tmp.S show.c -o tmp")
    if ret != 0:
        return ret
    ret = os.system("./tmp > want.txt 2>&1")
    if ret != 0:
        return ret
    return os.system("/usr/local/bin/valgrind --tool=none -q ./tmp > out.txt 2>&1")


def diff():
    with open("want.txt", "r") as want:
        with open("out.txt", "r") as out:
            lines1 = want.readlines()
            lines2 = out.readlines()
            if len(lines1) != len(lines2):
                print("len1 != len2")
                return False
            for i in range(0, len(lines1)):
                if i == 3 or i == 11 or i == 21:
                    continue
                if lines1[i] != lines2[i]:
                    print("want: " + lines1[i] + "out: " + lines2[i])
                    return False
            return True

def test(name, func, n):
    for _ in range(0, n):
        line = func(name)
        if build(line) != 0:
            print("Build failed!")
            return False
        if not diff():
            return False
    return True


def rand_reg():
    reg = np.random.randint(4, 32, 1, np.int32)
    while reg[0] == 11 or reg[0] == 21:
        reg = np.random.randint(4, 32, 1, np.int32)
    num = np.random.randint(np.iinfo(np.int64).min, np.iinfo(np.int64).max, 1, np.int64)
    return (reg[0], num[0])

def rand_reg_32():
    reg = np.random.randint(4, 32, 1, np.int32)
    while reg[0] == 11 or reg[0] == 21:
        reg = np.random.randint(4, 32, 1, np.int32)
    num = np.random.randint(0, np.iinfo(np.int32).max, 1, np.int32)
    return (reg[0], num[0])

def rand_imm(n, sign):
    min = 0
    max = 1 << n
    if (sign):
        min = -(1 << (n - 1))
        max = 1 << (n - 1)
    imm = np.random.randint(min, max, 1, np.int32)
    return imm[0]

def rand_imm2(n):
    imm = np.random.randint(0, 1 << n, 2, np.int32)
    min = imm[0]
    max = imm[1]
    if imm[1] < min:
        max = imm[0]
        min = imm[1]
    return (min, max)

def rd_rj(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    line += f"{name} $r{rd}, $r{rj}\n"
    return line

def rd_rj_rk(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    rk, v3 = rand_reg()
    line += f"li.d $r{rk}, {v3}\n    "
    line += f"{name} $r{rd}, $r{rj}, $r{rk}\n"
    return line

def rd_rj_rk_32(name):
    line = ""
    rd, v1 = rand_reg_32()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg_32()
    line += f"li.d $r{rj}, {v2}\n    "
    rk, v3 = rand_reg_32()
    line += f"li.d $r{rk}, {v3}\n    "
    line += f"{name} $r{rd}, $r{rj}, $r{rk}\n"
    return line

def rd_rj_si12(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    si12 = rand_imm(12, True)
    line += f"{name} $r{rd}, $r{rj}, {si12}\n"
    return line

def rd_rj_ui12(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    ui12 = rand_imm(12, False)
    line += f"{name} $r{rd}, $r{rj}, {ui12}\n"
    return line

def rd_rj_si16(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    si16 = rand_imm(16, True)
    line += f"{name} $r{rd}, $r{rj}, {si16}\n"
    return line

def rd_rj_ui5(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    ui5 = rand_imm(5, False)
    line += f"{name} $r{rd}, $r{rj}, {ui5}\n"
    return line

def rd_rj_ui6(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    ui6 = rand_imm(6, False)
    line += f"{name} $r{rd}, $r{rj}, {ui6}\n"
    return line

def rd_rj_msbw_lsbw(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    lsbw, msbw = rand_imm2(5)
    line += f"{name} $r{rd}, $r{rj}, {msbw}, {lsbw}\n"
    return line

def rd_rj_msbd_lsbd(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    lsbd, msbd = rand_imm2(6)
    line += f"{name} $r{rd}, $r{rj}, {msbd}, {lsbd}\n"
    return line

def rd_rj_rk_sa2(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    rk, v3 = rand_reg()
    line += f"li.d $r{rk}, {v3}\n    "
    sa2 = rand_imm(2, False)
    line += f"{name} $r{rd}, $r{rj}, $r{rk}, {sa2}\n"
    return line

def rd_rj_rk_sa2_add1(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    rk, v3 = rand_reg()
    line += f"li.d $r{rk}, {v3}\n    "
    sa2 = rand_imm(2, False) + 1
    line += f"{name} $r{rd}, $r{rj}, $r{rk}, {sa2}\n"
    return line

def rd_rj_rk_sa3(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    rj, v2 = rand_reg()
    line += f"li.d $r{rj}, {v2}\n    "
    rk, v3 = rand_reg()
    line += f"li.d $r{rk}, {v3}\n    "
    sa3 = rand_imm(3, False)
    line += f"{name} $r{rd}, $r{rj}, $r{rk}, {sa3}\n"
    return line

def rd_si20(name):
    line = ""
    rd, v1 = rand_reg()
    line += f"li.d $r{rd}, {v1}\n    "
    si20 = rand_imm(20, True)
    line += f"{name} $r{rd}, {si20}\n"
    return line

insts = [
    { "name": "add.w",      "func": rd_rj_rk            },
    { "name": "add.d",      "func": rd_rj_rk            },
    { "name": "sub.w",      "func": rd_rj_rk            },
    { "name": "sub.d",      "func": rd_rj_rk            },
    { "name": "slt",        "func": rd_rj_rk            },
    { "name": "sltu",       "func": rd_rj_rk            },
    { "name": "slti",       "func": rd_rj_si12          },
    { "name": "sltui",      "func": rd_rj_si12          },
    { "name": "nor",        "func": rd_rj_rk            },
    { "name": "and",        "func": rd_rj_rk            },
    { "name": "or",         "func": rd_rj_rk            },
    { "name": "xor",        "func": rd_rj_rk            },
    { "name": "orn",        "func": rd_rj_rk            },
    { "name": "andn",       "func": rd_rj_rk            },
    { "name": "mul.w",      "func": rd_rj_rk            },
    { "name": "mulh.w",     "func": rd_rj_rk            },
    { "name": "mulh.wu",    "func": rd_rj_rk            },
    { "name": "mul.d",      "func": rd_rj_rk            },
    { "name": "mulh.d",     "func": rd_rj_rk            },
    { "name": "mulh.du",    "func": rd_rj_rk            },
    { "name": "mulw.d.w",   "func": rd_rj_rk            },
    { "name": "mulw.d.wu",  "func": rd_rj_rk            },
    { "name": "div.w",      "func": rd_rj_rk_32         },
    { "name": "mod.w",      "func": rd_rj_rk_32         },
    { "name": "div.wu",     "func": rd_rj_rk_32         },
    { "name": "mod.wu",     "func": rd_rj_rk_32         },
    { "name": "div.d",      "func": rd_rj_rk            },
    { "name": "mod.d",      "func": rd_rj_rk            },
    { "name": "div.du",     "func": rd_rj_rk            },
    { "name": "mod.du",     "func": rd_rj_rk            },
    { "name": "alsl.w",     "func": rd_rj_rk_sa2_add1   },
    { "name": "alsl.wu",    "func": rd_rj_rk_sa2_add1   },
    { "name": "alsl.d",     "func": rd_rj_rk_sa2_add1   },
    { "name": "lu12i.w",    "func": rd_si20             },
    { "name": "lu32i.d",    "func": rd_si20             },
    { "name": "lu52i.d",    "func": rd_rj_si12          },
    { "name": "addi.w",     "func": rd_rj_si12          },
    { "name": "addi.d",     "func": rd_rj_si12          },
    { "name": "addu16i.d",  "func": rd_rj_si16          },
    { "name": "andi",       "func": rd_rj_ui12          },
    { "name": "ori",        "func": rd_rj_ui12          },
    { "name": "xori",       "func": rd_rj_ui12          },
    { "name": "sll.w",      "func": rd_rj_rk            },
    { "name": "srl.w",      "func": rd_rj_rk            },
    { "name": "sra.w",      "func": rd_rj_rk            },
    { "name": "sll.d",      "func": rd_rj_rk            },
    { "name": "srl.d",      "func": rd_rj_rk            },
    { "name": "sra.d",      "func": rd_rj_rk            },
    { "name": "rotr.w",     "func": rd_rj_rk            },
    { "name": "rotr.d",     "func": rd_rj_rk            },
    { "name": "slli.w",     "func": rd_rj_ui5           },
    { "name": "slli.d",     "func": rd_rj_ui6           },
    { "name": "srli.w",     "func": rd_rj_ui5           },
    { "name": "srli.d",     "func": rd_rj_ui6           },
    { "name": "srai.w",     "func": rd_rj_ui5           },
    { "name": "srai.d",     "func": rd_rj_ui6           },
    { "name": "rotri.w",    "func": rd_rj_ui5           },
    { "name": "rotri.d",    "func": rd_rj_ui6           },
    { "name": "ext.w.h",    "func": rd_rj               },
    { "name": "ext.w.b",    "func": rd_rj               },
    { "name": "clo.w",      "func": rd_rj               },
    { "name": "clz.w",      "func": rd_rj               },
    { "name": "cto.w",      "func": rd_rj               },
    { "name": "ctz.w",      "func": rd_rj               },
    { "name": "clo.d",      "func": rd_rj               },
    { "name": "clz.d",      "func": rd_rj               },
    { "name": "cto.d",      "func": rd_rj               },
    { "name": "ctz.d",      "func": rd_rj               },
    { "name": "revb.2h",    "func": rd_rj               },
    { "name": "revb.4h",    "func": rd_rj               },
    { "name": "revb.2w",    "func": rd_rj               },
    { "name": "revb.d",     "func": rd_rj               },
    { "name": "revh.2w",    "func": rd_rj               },
    { "name": "revh.d",     "func": rd_rj               },
    { "name": "bitrev.4b",  "func": rd_rj               },
    { "name": "bitrev.8b",  "func": rd_rj               },
    { "name": "bitrev.w",   "func": rd_rj               },
    { "name": "bitrev.d",   "func": rd_rj               },
    { "name": "bytepick.w", "func": rd_rj_rk_sa2        },
    { "name": "bytepick.d", "func": rd_rj_rk_sa3        },
    { "name": "maskeqz",    "func": rd_rj_rk            },
    { "name": "masknez",    "func": rd_rj_rk            },
    { "name": "bstrins.w",  "func": rd_rj_msbw_lsbw     },
    { "name": "bstrpick.w", "func": rd_rj_msbw_lsbw     },
    { "name": "bstrins.d",  "func": rd_rj_msbd_lsbd     },
    { "name": "bstrpick.d", "func": rd_rj_msbd_lsbd     },
]
n = 10
for inst in insts:
    if not test(inst["name"], inst["func"], n):
        print(f"{inst['name']} failed")
        break
    else:
        print(f"{inst['name']} passed")
```

同理，对浮点指令的测试只需要稍微修改脚本：

```python
def write(line, file):
    with open("dump.S", "r") as input:
        text = input.read()
        with open(file, "w") as output:
            output.write(text.replace("nop\n", line))


def build(line):
    write(line, "tmp.S")
    ret = os.system("gcc -nostdlib -static tmp.S show.c data.c -o tmp")
    if ret != 0:
        return ret
    ret = os.system("./tmp > want.txt 2>&1")
    if ret != 0:
        return ret
    return os.system("/usr/local/bin/valgrind --tool=none -q ./tmp > out.txt 2>&1")


def diff(high):
    with open("want.txt", "r") as want:
        with open("out.txt", "r") as out:
            lines1 = want.readlines()
            lines2 = out.readlines()
            if len(lines1) != len(lines2):
                print("len1 != len2")
                return False
            for i in range(0, len(lines1)):
                if high and lines1[i] != lines2[i]:
                    print("64-bit")
                    print("want: " + lines1[i] + "out: " + lines2[i])
                    return False
                elif lines1[14:] != lines2[14:]:
                    print("32-bit")
                    print("want: " + lines1[14:] + "out: " + lines2[14:])
                    return False
            return True


def test(name, func, n):
    for _ in range(0, n):
        line, high = func(name, n)
        if build(line) != 0:
            print("Build failed!")
            return False
        if not diff(high):
            return False
    return True


def fd_fj_s(name, n):
    line = "la.local $t0, fj_s\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, False


def fd_fj_d(name, n):
    line = "la.local $t0, fj_d\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, True


def fd_fj_fk_s(name, n):
    line = "la.local $t0, fj_s\n    "
    line += "la.local $t1, fk_s\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"fld.s $f2, $t1, {4 * n}\n    "
    line += f"{name} $f0, $f1, $f2\n"
    return line, False


def fd_fj_fk_d(name, n):
    line = "la.local $t0, fj_d\n    "
    line += "la.local $t1, fk_d\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"fld.d $f2, $t1, {8 * n}\n    "
    line += f"{name} $f0, $f1, $f2\n"
    return line, True


def fd_fj_fk_fa_s(name, n):
    line = "la.local $t0, fj_s\n    "
    line += "la.local $t1, fk_s\n    "
    line += "la.local $t2, fa_s\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"fld.s $f2, $t1, {4 * n}\n    "
    line += f"fld.s $f3, $t2, {4 * n}\n    "
    line += f"{name} $f0, $f1, $f2, $f3\n"
    return line, False


def fd_fj_fk_fa_d(name, n):
    line = "la.local $t0, fj_d\n    "
    line += "la.local $t1, fk_d\n    "
    line += "la.local $t2, fa_d\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"fld.d $f2, $t1, {8 * n}\n    "
    line += f"fld.d $f3, $t2, {8 * n}\n    "
    line += f"{name} $f0, $f1, $f2, $f3\n"
    return line, True


def fd_fj_s_d(name, n):
    line = "la.local $t0, fj_d\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, False


def fd_fj_d_s(name, n):
    line = "la.local $t0, fj_s\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, True


def fd_fj_w_s(name, n):
    line = "la.local $t0, fj_s\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, False


def fd_fj_w_d(name, n):
    line = "la.local $t0, fj_d\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, False


def fd_fj_l_s(name, n):
    line = "la.local $t0, fj_s\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, True


def fd_fj_l_d(name, n):
    line = "la.local $t0, fj_d\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, True


def fd_fj_s_w(name, n):
    line = "la.local $t0, fj_w\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, False


def fd_fj_s_l(name, n):
    line = "la.local $t0, fj_l\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, False


def fd_fj_d_w(name, n):
    line = "la.local $t0, fj_w\n    "
    line += f"fld.s $f1, $t0, {4 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, True


def fd_fj_d_l(name, n):
    line = "la.local $t0, fj_l\n    "
    line += f"fld.d $f1, $t0, {8 * n}\n    "
    line += f"{name} $f0, $f1\n"
    return line, True


insts = [
    { "name": "fadd.s",         "func": fd_fj_fk_s },
    { "name": "fadd.d",         "func": fd_fj_fk_d },
    { "name": "fsub.s",         "func": fd_fj_fk_s },
    { "name": "fsub.d",         "func": fd_fj_fk_d },
    { "name": "fmul.s",         "func": fd_fj_fk_s },
    { "name": "fmul.d",         "func": fd_fj_fk_d },
    { "name": "fdiv.s",         "func": fd_fj_fk_s },
    { "name": "fdiv.d",         "func": fd_fj_fk_d },
    { "name": "fmadd.s",        "func": fd_fj_fk_fa_s },
    { "name": "fmadd.d",        "func": fd_fj_fk_fa_d },
    { "name": "fmsub.s",        "func": fd_fj_fk_fa_s },
    { "name": "fmsub.d",        "func": fd_fj_fk_fa_d },
    { "name": "fnmadd.s",       "func": fd_fj_fk_fa_s },
    { "name": "fnmadd.d",       "func": fd_fj_fk_fa_d },
    { "name": "fnmsub.s",       "func": fd_fj_fk_fa_s },
    { "name": "fnmsub.d",       "func": fd_fj_fk_fa_d },
    { "name": "fmax.s",         "func": fd_fj_fk_s },
    { "name": "fmax.d",         "func": fd_fj_fk_d },
    { "name": "fmin.s",         "func": fd_fj_fk_s },
    { "name": "fmin.d",         "func": fd_fj_fk_d },
    { "name": "fmaxa.s",        "func": fd_fj_fk_s },
    { "name": "fmaxa.d",        "func": fd_fj_fk_d },
    { "name": "fmina.s",        "func": fd_fj_fk_s },
    { "name": "fmina.d",        "func": fd_fj_fk_d },
    { "name": "fabs.s",         "func": fd_fj_s },
    { "name": "fabs.d",         "func": fd_fj_d },
    { "name": "fneg.s",         "func": fd_fj_s },
    { "name": "fneg.d",         "func": fd_fj_d },
    { "name": "fsqrt.s",        "func": fd_fj_s },
    { "name": "fsqrt.d",        "func": fd_fj_d },
    { "name": "frecip.s",       "func": fd_fj_s },
    { "name": "frecip.d",       "func": fd_fj_d },
    { "name": "frsqrt.s",       "func": fd_fj_s },
    { "name": "frsqrt.d",       "func": fd_fj_d },
    { "name": "fscaleb.s",      "func": fd_fj_fk_s },
    { "name": "fscaleb.d",      "func": fd_fj_fk_d },
    { "name": "flogb.s",        "func": fd_fj_s },
    { "name": "flogb.d",        "func": fd_fj_d },
    { "name": "fcopysign.s",    "func": fd_fj_fk_s },
    { "name": "fcopysign.d",    "func": fd_fj_fk_d },
    { "name": "fclass.s",       "func": fd_fj_s },
    { "name": "fclass.d",       "func": fd_fj_d },
    { "name": "fcvt.s.d",       "func": fd_fj_s_d },
    { "name": "fcvt.d.s",       "func": fd_fj_d_s },
    { "name": "ftintrm.w.s",    "func": fd_fj_w_s },
    { "name": "ftintrm.w.d",    "func": fd_fj_w_d },
    { "name": "ftintrm.l.s",    "func": fd_fj_l_s },
    { "name": "ftintrm.l.d",    "func": fd_fj_l_d },
    { "name": "ftintrp.w.s",    "func": fd_fj_w_s },
    { "name": "ftintrp.w.d",    "func": fd_fj_w_d },
    { "name": "ftintrp.l.s",    "func": fd_fj_l_s },
    { "name": "ftintrp.l.d",    "func": fd_fj_l_d },
    { "name": "ftintrz.w.s",    "func": fd_fj_w_s },
    { "name": "ftintrz.w.d",    "func": fd_fj_w_d },
    { "name": "ftintrz.l.s",    "func": fd_fj_l_s },
    { "name": "ftintrz.l.d",    "func": fd_fj_l_d },
    { "name": "ftintrne.w.s",   "func": fd_fj_w_s },
    { "name": "ftintrne.w.d",   "func": fd_fj_w_d },
    { "name": "ftintrne.l.s",   "func": fd_fj_l_s },
    { "name": "ftintrne.l.d",   "func": fd_fj_l_d },
    { "name": "ftint.w.s",      "func": fd_fj_w_s },
    { "name": "ftint.w.d",      "func": fd_fj_w_d },
    { "name": "ftint.l.s",      "func": fd_fj_l_s },
    { "name": "ftint.l.d",      "func": fd_fj_l_d },
    { "name": "ffint.s.w",      "func": fd_fj_s_w },
    { "name": "ffint.s.l",      "func": fd_fj_s_l },
    { "name": "ffint.d.w",      "func": fd_fj_d_w },
    { "name": "ffint.d.l",      "func": fd_fj_d_l },
    { "name": "frint.s",        "func": fd_fj_s },
    { "name": "frint.d",        "func": fd_fj_d }
]

n = 24
for inst in insts:
    if not test(inst["name"], inst["func"], n):
        print(f"{inst['name']} failed")
        break
    else:
        print(f"{inst['name']} passed")
```

在 `dump.S` 前加入对 `fcsr` 寄存器的修改：

```text
.text
.globl _start
_start:
    // li.w       $t0, 0x0
    // li.w       $t0, 0x100
    // li.w       $t0, 0x200
    li.w       $t0, 0x300
    movgr2fcsr $r0, $t0

    nop
    ...
```

使用的数据文件 `data.c` 如下（抄的 mips 测试案例）：

```c
const float fj_s[] = {
    0,         456.25,   3,          -1,
    1384.5,    -7.25,    1000000000, -5786.5,
    1752,      0.015625, 0.03125,    -248562.75,
    -45786.5,  456,      34.03125,   45786.75,
    1752065,   107,      -45667.25,  -7,
    -347856.5, 356047.5, -1.0,       23.0625
};

const double fj_d[] = {
    0,         456.25,   3,          -1,
    1384.5,    -7.25,    1000000000, -5786.5,
    1752,      0.015625, 0.03125,    -248562.75,
    -45786.5,  456,      34.03125,   45786.75,
    1752065,   107,      -45667.25,  -7,
    -347856.5, 356047.5, -1.0,       23.0625
};

const float fk_s[] = {
    -4578.5, 456.25,   34.03125, 4578.75,
    175,     107,      -456.25,  -7.25,
    -3478.5, 356.5,    -1.0,     23.0625,
    0,       456.25,   3,        -1,
    1384.5,  -7,       100,      -5786.5,
    1752,    0.015625, 0.03125,  -248562.75
};

const double fk_d[] = {
    -45786.5,  456.25,   34.03125,   45786.75,
    1752065,   107,      -45667.25,  -7.25,
    -347856.5, 356047.5, -1.0,       23.0625,
    0,         456.25,   3,          -1,
    1384.5,    -7,       1000000000, -5786.5,
    1752,      0.015625, 0.03125,    -248562.75
};

const float fa_s[] = {
    -347856.5,  356047.5,  -1.0,       23.0625,
    1752,       0.015625,  0.03125,    -248562.75,
    1384.5,     -7.25,     1000000000, -5786.5,
    -347856.75, 356047.75, -1.0,       23.03125,
    0,          456.25,    3,          -1,
    -45786.5,   456,       34.03125,   45786.03125,
};

const double fa_d[] = {
    -347856.5,  356047.5,  -1.0,       23.0625,
    1752,       0.015625,  0.03125,    -248562.75,
    1384.5,     -7.25,     1000000000, -5786.5,
    -347856.75, 356047.75, -1.0,       23.03125,
    0,          456.25,    3,          -1,
    -45786.5,   456,       34.03125,   45786.03125,
};

const int fj_w[] = {
    0,          456,        3,          -1,
    0xffffffff, 356,        1000000000, -5786,
    1752,       24575,      10,         -248562,
    -45786,     456,        34,         45786,
    1752065,    107,        -45667,     -7,
    -347856,    0x80000000, 0xfffffff,  23,
};

const long fj_l[] = {
    18,         25,         3,          -1,
    0xffffffff, 356,        1000000,    -5786,
    -1,         24575,      10,         -125458,
    -486,       456,        34,         45786,
    0,          1700000,    -45667,     -7,
    -347856,    0x80000000, 0xfffffff,  23,
};
```

对于 `show.c`，只检查 `f0` 和 `fcsr` 即可：

```c
...

int show(unsigned long *regs)
{
    char s[20], t[20];
    my_puts("f");
    my_itoa(s, 0, 10);
    my_puts(s);
    my_puts(":\t");
    my_itoa(s, regs[32], 16);
    widen(t, s, 16);
    my_puts(t);
    my_puts("\n");
    my_puts("fcsr");
    my_itoa(s, 0, 10);
    my_puts(s);
    my_puts(":\t");
    my_itoa(s, regs[64], 16);
    widen(t, s, 8);
    my_puts(t);
    my_puts("\n");
    return 0;
}
```

借助这个脚本我发现了大量翻译指令时的笔误……

### 比对文件输出

当时遇到一个问题，发现运行动态链接程序会发生段错误，而静态链接的版本能正常跑过。

我希望能通过比对运行动态和静态链接两个版本的 Valgrind 日志，来定位是哪条或者哪些指令可能出了问题。
于是顺手写了一个 golang 的比较程序：

```golang
package main

import (
    "io/ioutil"
    "log"
    "regexp"
    "sort"
    "strings"
)

var insnRe *regexp.Regexp
var insnRe2 *regexp.Regexp

func unique(s []string) []string {
    sort.Strings(s)
    res := []string{}
    for i := 0; i < len(s); i++ {
        if (i > 0 && s[i-1] == s[i]) || len(s[i]) == 0 {
            continue
        }
        res = append(res, s[i])
    }
    return res
}

func findMatch(lines []string, re *regexp.Regexp) []string {
    insn := []string{}
    for _, line := range lines {
        found := re.FindAllStringSubmatch(line, -1)
        if len(found) == 1 {
            insn = append(insn, found[0][1])
        }
    }
    return insn
}

func read(file string) ([]string, error) {
    text, err := ioutil.ReadFile(file)
    if err != nil {
        return nil, err
    }
    lines := strings.Split(string(text), "\n")
    return lines, nil
}

func diff(good, bad []string) []string {
    good = unique(good)
    bad = unique(bad)
    res := []string{}
    i, j := 0, 0
    for i < len(good) && j < len(bad) {
        if good[i] == bad[j] {
            i++
            j++
        } else if good[i] < bad[j] {
            // res = append(res, "-"+good[i])
            i++
        } else {
            // res = append(res, "+"+bad[j])
            res = append(res, bad[j])
            j++
        }
    }
    for ; i < len(good); i++ {
        // res = append(res, "-"+good[i])
    }
    for ; j < len(bad); j++ {
        // res = append(res, "+"+bad[j])
        res = append(res, bad[j])
    }
    return res
}

func compare(good, bad []string, re *regexp.Regexp) string {
    good = findMatch(good, re)
    bad = findMatch(bad, re)
    text := diff(good, bad)
    sort.Strings(text)
    return strings.Join(text, "\n")
}

func main() {
    insnRe = regexp.MustCompile(`\t0x[A-Z0-9]+:\t0x[A-Z0-9]+\t(.*)`)
    insnRe2 = regexp.MustCompile(`^\s*\d+\s+(.*)`)
    good, err := read("good.txt")
    if err != nil {
        log.Fatalln(err)
    }
    bad, err := read("bad.txt")
    if err != nil {
        log.Fatalln(err)
    }
    out := compare(good, bad, insnRe)
    out += "\n\n"
    out += compare(good, bad, insnRe2)
    out += "\n"
    err = ioutil.WriteFile("diff.txt", []byte(out), 0644)
    if err != nil {
        log.Fatalln(err)
    }
}
```

## 检查笔误、遗漏

检查简单的笔误，以及博文里指令是否写全：

```golang
package main

import (
    "fmt"
    "io/ioutil"
    "log"
    "regexp"
    "sort"
    "strings"
)

func find(re *regexp.Regexp, line string, set map[string]bool) {
    found := re.FindAllString(line, -1)
    for _, s := range found {
        set[s] = true
    }
}

func show(set map[string]bool) []string {
    list := []string{}
    for s, _ := range set {
        list = append(list, s)
    }
    sort.Strings(list)
    res := []string{}
    for _, l := range list {
        l = strings.ReplaceAll(l, "IRConst", "Ico")
        l = strings.ReplaceAll(l, "IRExpr", "Iex")
        l = strings.ReplaceAll(l, "IRStmt", "Ist")
        res = append(res, l)
    }
    return res
}

func findAll(text string) []string {
    lines := strings.Split(text, "\n")
    stRe := regexp.MustCompile(`(IRStmt_|Ist_)\w+`)
    exRe := regexp.MustCompile(`(IRExpr_|Iex_)\w+`)
    opRe := regexp.MustCompile(`Iop_\w+`)
    jkRe := regexp.MustCompile(`Ijk_\w+`)
    coRe := regexp.MustCompile(`(IRConst_|Ico_)\w+`)
    tyRe := regexp.MustCompile(`Ity_\w+`)
    enRe := regexp.MustCompile(`Iend_\w+`)
    stSet := make(map[string]bool)
    exSet := make(map[string]bool)
    opSet := make(map[string]bool)
    jkSet := make(map[string]bool)
    coSet := make(map[string]bool)
    tySet := make(map[string]bool)
    enSet := make(map[string]bool)
    for _, line := range lines {
        find(stRe, line, stSet)
        find(exRe, line, exSet)
        find(opRe, line, opSet)
        find(jkRe, line, jkSet)
        find(coRe, line, coSet)
        find(tyRe, line, tySet)
        find(enRe, line, enSet)
    }
    res := []string{}
    res = append(res, show(enSet)...)
    res = append(res, show(coSet)...)
    res = append(res, show(tySet)...)
    res = append(res, show(opSet)...)
    res = append(res, show(exSet)...)
    res = append(res, show(jkSet)...)
    res = append(res, show(stSet)...)
    return res
}

func diff2(out1, out2 []string) {
    i, j := 0, 0
    for i < len(out1) && j < len(out2) {
        if out1[i] == out2[j] {
            i++
            j++
        } else if out1[i] < out2[j] {
            fmt.Println("-" + out1[i])
            i++
        } else {
            fmt.Println("+" + out2[j])
            j++
        }
    }
    for ; i < len(out1); i++ {
        fmt.Println("-" + out1[i])
    }
    for ; j < len(out2); j++ {
        fmt.Println("+" + out2[j])
    }
}

func check_name(text string) {
    lines := strings.Split(text, "\n")
    level := 0
    name := ""
    prefix := "static Bool gen_"
    prefix2 := "   DIP(\""
    prefix3 := "   UInt "
    for _, line := range lines {
        if len(line) == 0 {
            continue
        }
        if strings.Contains(line, "Disassemble a single LOONGARCH64 instruction") {
            break
        }
        if strings.Contains(line, "{") {
            level++
        }
        if strings.Contains(line, "}") {
            level--
        }
        if level == 0 && strings.HasPrefix(line, prefix) {
            pos := strings.Index(line, " (")
            if pos < len(prefix) {
                log.Fatalln("prefix " + line)
            }
            name = line[len(prefix):pos]
        }
        if level == 1 && strings.HasPrefix(line, prefix2) {
            pos := strings.Index(line, " %")
            if pos < len(prefix2) {
                log.Fatalln("prefix2 " + line)
            }
            name2 := line[len(prefix2):pos]
            if strings.ReplaceAll(name2, ".", "_") != name {
                fmt.Println("check: " + name + " != " + name2)
            }
        }
        if level == 1 && strings.HasPrefix(line, prefix3) {
            pos1 := strings.Index(line, " =")
            pos2 := strings.Index(line, "_")
            pos3 := strings.Index(line, "(insn)")
            if pos1 > 0 && pos2 > 0 && pos3 > 0 {
                name3 := strings.TrimSpace(line[8:pos1])
                name4 := strings.TrimSpace(line[pos2+1 : pos3])
                if name3 != name4 &&
                    name3 != "hint" && name3 != "fcsr" &&
                    name3 != "lsb" && name3 != "msb" {
                    fmt.Println("check in " + name + " : " + name3 + " != " + name4)
                }
            }
        }
    }
}

func check_dup(text string) {
    lines := strings.Split(text, "\n")
    level := 0
    names := []string{
        "getIReg8(rd)", "getIReg16(rd)", "getIReg32(rd)", "getIReg64(rd)",
        "getIReg8(rj)", "getIReg16(rj)", "getIReg32(rj)", "getIReg64(rj)",
        "getIReg8(rk)", "getIReg16(rk)", "getIReg32(rk)", "getIReg64(rk)",
        "getFReg8(fd)", "getFReg16(fd)", "getFReg32(fd)", "getFReg64(fd)",
        "getFReg8(fj)", "getFReg16(fj)", "getFReg32(fj)", "getFReg64(fj)",
        "getFReg8(fk)", "getFReg16(fk)", "getFReg32(fk)", "getFReg64(fk)",
        "getFReg8(fa)", "getFReg16(fa)", "getFReg32(fa)", "getFReg64(fa)",
    }
    set := make(map[int]bool)
    prefix := "static Bool gen_"
    name := ""
    for _, line := range lines {
        if len(line) == 0 {
            continue
        }
        if strings.Contains(line, "{") {
            level++
        } else if strings.Contains(line, "}") {
            level--
        }
        if level == 0 {
            if strings.HasPrefix(line, prefix) {
                pos := strings.Index(line, " (")
                if pos < len(prefix) {
                    log.Fatalln("prefix")
                }
                name = line[len(prefix):pos]
            }
            for k := range set {
                delete(set, k)
            }
        }
        if level == 1 {
            for i, n := range names {
                if strings.Contains(line, n) {
                    if set[i] {
                        fmt.Println("check: " + n + " in " + name)
                    } else {
                        set[i] = true
                    }
                }
            }
        }
    }
}

func main() {
    filename := "xxx/valgrind/VEX/priv/guest_loongarch64_toIR.c"
    filename2 := "xxx/blog/content/zh-cn/posts/valgrind/loongarch.md"
    text1, _ := ioutil.ReadFile(filename)
    check_name(string(text1))
    check_dup(string(text1))
    out1 := findAll(string(text1))
    text2, _ := ioutil.ReadFile(filename2)
    out2 := findAll(string(text2))
    diff2(out1, out2)
}
```
