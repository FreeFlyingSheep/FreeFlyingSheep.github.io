---
title: "LoongArch 汇编"
date: 2022-07-19
description: ""
menu:
  sidebar:
    name: "LoongArch 汇编"
    identifier: loongarch-assembly
    parent: loongarch
    weight: 100
tags: ["LoongArch", "汇编"]
categories: ["LoongArch"]
---

简单介绍 LoongArch 的汇编指令和一些 psABI (Processor-Specific Application Binary Interface) 规范，相关文档见[文档仓库](https://github.com/loongson/LoongArch-Documentation)。

<!--more-->

## 寄存器

LoongArch 的寄存器使用约定基本和 RISC-V 相同。

### 通用寄存器

| 名称 | 助记符 | 含义 | 保存者 |
| --- | --- | --- | --- |
| `$r0` | `$zero` | 常数零 | - |
| `$r1` | `$ra` | 返回地址 | 被调用者保存 |
| `$r2` | `$tp` | 线程指针 | - |
| `$r3` | `$sp` | 栈指针 | 调用者保存 |
| `$r4`-`$r5` | `$a0`-`$a1` | 参数/返回值寄存器 | 被调用者保存 |
| `$r6`-`$r11` | `$a2`-`$a7` | 参数寄存器 | 被调用者保存 |
| `$r12`-`$r20` | `$t0`-`$t8` | 临时寄存器 | 被调用者保存 |
| `$r21` | - | 保留 | - |
| `$r22` | `$fp`/`$s9` | 帧指针/静态寄存器 | 调用者保存 |
| `$r23`-`$r31` | `$s0`-`$s8` | 静态寄存器 | 调用者保存 |

### `PC` 寄存器

`PC` 寄存器记录当前指令的地址，因此总是 4 字节对齐的，它只能被指令间接修改。

### 浮点寄存器

| 名称 | 助记符 | 含义 | 保存者 |
| --- | --- | --- | --- |
| `$f0`-`$f1` | `$fa0`-`$fa1` | 参数/返回值寄存器 | 被调用者保存 |
| `$f2`-`$f7` | `$fa2`-`$fa7` | 参数寄存器 | 被调用者保存 |
| `$f8`-`$f23` | `$ft0`-`$ft15` | 临时寄存器 | 被调用者保存 |
| `$f24`-`$f31` | `$fs0`-`$fs7` | 静态寄存器 | 调用者保存 |

### 条件标志寄存器

`$fcc0`-`$fcc7` 一共 8 个条件标志寄存器（CFR），每个长度都是 1 位，可以读写。

### 浮点控制状态寄存器

`$fcsr0`-`$fcsr3` 一共 4 个浮点控制状态寄存器（FCSR），每个长度都是 32 位，可以读写，具体功能可以参考手册（准确地说，只有一个 `$fcsr0` 寄存器，`$fcsr1`-`$fcsr3` 是 `$fcsr0` 中部分域的别名）。

值得注意的是，部分保留的位域确实被用于其他用途了，写操作有效，读未必返回 `0`，比如第 8、 9 位（而且测试发现它们属于 `$fcsr1` 的一部分），但具体作用还未公开。

## 指令格式

这是我觉得最蛋疼的地方，手册中指出 LoongArch 有 9 种指令格式，但在实际开发中（特别是需要汇编、反汇编的时候），我们往往需要一个更具体的划分，有大佬划分出了 [39 种指令格式](https://blog.xen0n.name/zh-cn/posts/tinkering/loongarch-faq/#真实情况)。
好在指令格式对通常意义上的编写汇编代码基本透明。

## 基础指令

### 汇编指令格式

补充说明以下几点：

- LoongArch 是小端序的。
- 此处只考虑 64 位指令集，即 LoongArch64。
- 整数数据类型分为 5 种：比特（1 位，记为 "b"）、字节（8 位，记为 "B"）、半字（16 位，记为 "H"）、字（32 位，记为 "W"）、双字（64 位，记为 "D"）。
- 浮点数据类型分为 2 种：单精度（32 位，记为 "S"）、双精度（64 位，记为 "D"）。
- 定点数据类型分为 2 种（这是在浮点转换指令中用到的）：字（32 位，记为 "W"）、长字（64 位，记为 "L"）。

手册卷一总共包含 374 条指令，其中基础整数指令有 203 条，基础浮点指令有 147 条（这里统计时把 `fcmp.cond.s` 和 `fcmp.cond.d` 的 22 种 `cond` 展开了），特权指令有 24 条。
这里我只介绍基础指令，因为特权指令只可能在开发内核之类的软件时被用到。

大部分汇编指令遵循下面的格式：

- 一元运算指令：`指令名 目的操作数, 源操作数`
- 二元运算指令：`指令名 目的操作数, 源操作数1, 源操作数2`
- 三元运算指令：`指令名 目的操作数, 源操作数1, 源操作数2, 源操作数3`
- 加载指令：`指令名 目的操作数, 基址, 偏移`
- 存储指令：`指令名 源操作数, 基址, 偏移`

可以根据指令名后缀来判断操作数类型。
在之后的指令介绍中，统一使用小写字母，方便阅读。
不需要担心 "b" 和 "B" 冲突，因为指令后缀只有字节（"B"），没有比特（"b"）。

操作数如果是立即数，通常会在指令名里有体现，比如带 "i"。
同样的，如果是无符号数，指令名通常会带 "u"。

对于满足上述格式的指令，看字面意思就知道是干嘛的，我不会再展开介绍。
之后提到寄存器时，比如 `rj`，指的是 `rj` 寄存器里的数值，而不是寄存器编号。

### 基础整数指令

#### 加法

- `add.w rd, rj, rk`
- `add.d rd, rj, rk`
- `addi.w rd, rj, si12`
- `addi.d rd, rj, si12`
- `addu16i.d rd, rj, si16`

其中，`addu16i.d` 是把 16 位立即数 `si16` 左移 16 位后符号扩展，再加上 `rj` 的值，放入 `rd`。

#### 减法

- `sub.w rd, rj, rk`
- `sub.d rd, rj, rk`

#### 条件置位

- `slt rd, rj, rk`
- `sltu rd, rj, rk`
- `slti rd, rj, si12`
- `sltui rd, rj, si12`

"slt" 推测为 "set if less than" 的缩写。

#### 位运算

- `nor rd, rj, rk`
- `and rd, rj, rk`
- `or rd, rj, rk`
- `xor rd, rj, rk`
- `orn rd, rj, rk`
- `andn rd, rj, rk`
- `andi rd, rj, ui12`
- `ori rd, rj, ui12`
- `xori rd, rj, ui12`

其中，`nor` 是 `rd = ~(rj | rk)`，`orn` 是 `rd = rj | (~rk)`，`andn` 是 `rd = rj & (~rk)`。

#### 乘法

- `mul.w rd, rj, rk`
- `mulh.w rd, rj, rk`
- `mulh.wu rd, rj, rk`
- `mul.d rd, rj, rk`
- `mulh.d rd, rj, rk`
- `mulh.du rd, rj, rk`
- `mulw.d.w rd, rj, rk`
- `mulw.d.wu rd, rj, rk`

#### 除法

- `div.w rd, rj, rk`
- `mod.w rd, rj, rk`
- `div.wu rd, rj, rk`
- `mod.wu rd, rj, rk`
- `div.d rd, rj, rk`
- `mod.d rd, rj, rk`
- `div.du rd, rj, rk`
- `mod.du rd, rj, rk`

`div.w`、`div.wu`、`mod.w` 和 `mod.wu` 需要特别注意，如果除数和被除数超过了 32 位有符号数的数值范围，那么结果是未定义的。
这里的“超过 32 位有符号数的数值范围”是指，负数高 32 位必须全为 `1`，正数高 32 位必须全为 `0`。

除法指令不会产生异常，即使除数为 `0`，这时候结果未定义。

#### 左移后加法

- `alsl.w rd, rj, rk, sa2`
- `alsl.wu rd, rj, rk, sa2`
- `alsl.d rd, rj, rk, sa2`

"alsl" 是 "add logical shift left" 的缩写，这命名只能靠猜啊。

本质上是执行 `rd = (rj << sa2) + rk`，需要注意的是这里 `sa2` 的取值范围是 `[1, 4]`。
手册上是写的是 `rj << (sa2 + 1)`，这里非常容易引起误解，因为指令编码和我们需要写的汇编不同。
指令编码里的 `sa2` 的取值范围是 `[0, 3]`，而汇编器需要的 `sa2` 的取值范围却是 `[1, 4]`（即需要我们手动加一）。

#### 加载立即数相关运算

- `lu12i.w rd, si20`
- `lu32i.d rd, si20`
- `lu52i.d rd, rj, si12`

这几条指令往往和 `ori` 组合，用于[加载立即数](#加载立即数)。

#### `PC` 相关运算

- `pcaddi rd, si20`
- `pcalau12i rd, si20`
- `pcaddu12i rd, si20`
- `pcaddu18i rd, si20`

其中，"pcalau" 是 "pc align add upper" 的缩写，鬼才命名，反正我是猜不出来。

这些指令主要在[加载地址](#加载地址)时被使用。

#### 移位运算

- `sll.w rd, rj, rk`
- `srl.w rd, rj, rk`
- `sra.w rd, rj, rk`
- `sll.d rd, rj, rk`
- `srl.d rd, rj, rk`
- `sra.d rd, rj, rk`
- `rotr.w rd, rj, rk`
- `rotr.d rd, rj, rk`
- `slli.w rd, rj, ui5`
- `slli.d rd, rj, ui6`
- `srli.w rd, rj, ui5`
- `srli.d rd, rj, ui6`
- `srai.w rd, rj, ui5`
- `srai.d rd, rj, ui6`
- `rotri.w rd, rj, ui5`
- `rotri.d rd, rj, ui6`

以 `sll` 为例，"s" 是 "shift"，第一个 "l" 是 "left"，第二个 "l" 是 "logical"，代表逻辑左移；
`sra` 中的 "r" 是 "right"，"a" 是 "arithmetical"，代表算术右移；
`rotr` 中的 "rot" 是 "rotate"，代表循环右移；
以此类推。

#### 符号扩展

- `ext.w.h rd, rj`
- `ext.w.b rd, rj`

类似这种多个后缀的，按顺序对上，比如 `ext.w.h`，`rd`（目的操作数）是 `w`，`rj`（源操作数）是 `h`，那就是从 16 位符号扩展到 32 位。
同样的，`ext.w.b` 是从 8 位符号扩展到 32 位。

#### 统计 `0`、`1` 的个数

- `clo.w rd, rj`
- `clz.w rd, rj`
- `cto.w rd, rj`
- `ctz.w rd, rj`
- `clo.d rd, rj`
- `clz.d rd, rj`
- `cto.d rd, rj`
- `ctz.d rd, rj`

`clo` 是统计高位起始连续 `1` 的个数；
`clz` 是统计高位起始连续 `0` 的个数；
`cto` 是统计低位起始连续 `1` 的个数；
`ctz` 是统计低位起始连续 `0` 的个数。

#### 字节、半字、比特逆序

- `revb.2h rd, rj`
- `revb.4h rd, rj`
- `revb.2w rd, rj`
- `revb.d rd, rj`
- `revh.2w rd, rj`
- `revh.d rd, rj`
- `bitrev.4b rd, rj`
- `bitrev.8b rd, rj`
- `bitrev.w rd, rj`
- `bitrev.d rd, rj`

`revb` 是字节逆序，`revh` 是半字逆序，`bitrev` 是比特逆序；`2h` 即对两个半字分别进行逆序操作，`4h` 即对四个半字分别进行逆序操作，`2w` 即对两个字分别进行逆序操作，`d` 即对一个双字进行逆序操作，`4b` 即对 4 个字节进行逆序，`8b` 即对 8 个字节进行逆序。

以 `revb.2h` 为例，先对 `rj` 的低 32 位（一个半字）进行字节逆序（把 `[0, 15]` 位变成 `[8, 15] + [0, 7]`），再 `rj` 的高 32 位（另一个半字）进行字节逆序（把 `[16, 31]` 位变成 `[24, 31] + [16, 23]`），最终结果放入 `rd` 中；
以 `revh.d` 为例，对 64 位的 `rj` 进行半字逆序（把 `[0, 63]` 位变成 `[48, 63] + [32, 47] + [16, 31] + [0, 15]`），结果放入 `rd` 中；
以 `bitrev.4b` 为例，首先把 `rj` 的 `[0, 7]` 位逆序，其次把 `[8, 15]` 位逆序，然后把 `[16, 23]` 位逆序，接着把 `[24, 32]` 位逆序，最后把结果放入 `rd` 中。
以此类推。

#### 拼接

- `bytepick.w rd, rj, rk, sa2`
- `bytepick.d rd, rj, rk, sa3`

对于 `bytepick.w`，拼接后 `rd` 的低位为 `rj` 的 `[8 * (4 - sa2), 31]` 位，高位为 `rk` 的 `[0, 8 * (4 - sa2)]` 位。
对于 `bytepick.d`，拼接后 `rd` 的低位为 `rj` 的 `[8 * (8 - sa3), 63]` 位，高位为 `rk` 的 `[0, 8 * (8 - sa3)]` 位。

显然上述条件在边界时会越界，手册没有说明，但可以确定当 `sa2`、`sa3` 为 `0` 时，`rd = rk`；当  `sa2`、`sa3` 分别为 `4`、`8` 时，`rd = rj`。

#### 条件赋值

- `maskeqz rd, rj, rk`
- `masknez rd, rj, rk`

对于 `maskeqz`，`rd = (rk == 0) ? 0 : rj`；
对于 `masknez`，`rd = (rk != 0) ? 0 : rj`。

#### 替换

- `bstrins.w rd, rj, msbw, lsbw`
- `bstrins.d rd, rj, msbd, lsbd`

"bstrins" 是 "bit string insert" 的缩写，这个命名相对好猜一点。

把 `rd` 中的 `msbw - lsbw`/`msbd - lsbd` 替换为 `rj` 中的 `[0, msbw - lsbw]`/`[0, msbd - lsbd]` 位。
我们必须确保 `msbw`/`msbd` 大于 `lsbw`/`lsbd`。
手册中的伪代码在边界时会越界，处理情况参考上面的 `bytepick`。

#### 截取

- `bstrpick.w rd, rj, msbw, lsbw`
- `bstrpick.d rd, rj, msbd, lsbd`

把 `rj` 中的 `[0, msbw - lsbw]`/`[0, msbd - lsbd]` 位零扩展放入 `rd`。
同样的，我们必须确保 `msbw`/`msbd` 大于 `lsbw`/`lsbd`。

#### 访存

- `ld.b rd, rj, si12`
- `ld.h rd, rj, si12`
- `ld.w rd, rj, si12`
- `ld.d rd, rj, si12`
- `st.b rd, rj, si12`
- `st.h rd, rj, si12`
- `st.w rd, rj, si12`
- `st.d rd, rj, si12`
- `ld.bu rd, rj, si12`
- `ld.hu rd, rj, si12`
- `ld.wu rd, rj, si12`
- `ldx.b rd, rj, rk`
- `ldx.h rd, rj, rk`
- `ldx.w rd, rj, rk`
- `ldx.d rd, rj, rk`
- `stx.b rd, rj, rk`
- `stx.h rd, rj, rk`
- `stx.w rd, rj, rk`
- `stx.d rd, rj, rk`
- `ldx.bu rd, rj, rk`
- `ldx.hu rd, rj, rk`
- `ldx.wu rd, rj, rk`
- `ldptr.w rd, rj, si14`
- `stptr.w rd, rj, si14`
- `ldptr.d rd, rj, si14`
- `stptr.d rd, rj, si14`

"ld" 是 "load" 的缩写，"st" 是 "store" 的缩写。
带 "x" 的指令代表偏移用寄存器存储。
带 "u" 的指令代表加载的是无符号数。

`ldptr.w`/`ldptr.d`/`stptr.w`/`stptr.d` 和 `ld.w`/`ld.d`/`st.w`/`st.d` 的区别除了立即数是 14 位外，还会把立即数左移 2 位。
但在写汇编代码时，我们不需要考虑移位（比如 `ld.w rd, rj, 8` 和 `ldptr.w rd, rj, 8` 访问的是同一个地址），这又是一个汇编器和指令编码不同的地方。

上述访存指令在处理器不支持非对齐访问时，如果遇到非对齐，会触发 `SIGBUS` 异常。

#### 预取

- `preld hint, rj, si12`
- `preldx hint, rj, rk`

主要是为了将数据读取到缓存中，详细见手册。

#### 边界检查访存

- `ldgt.b rd, rj, rk`
- `ldgt.h rd, rj, rk`
- `ldgt.w rd, rj, rk`
- `ldgt.d rd, rj, rk`
- `ldle.b rd, rj, rk`
- `ldle.h rd, rj, rk`
- `ldle.w rd, rj, rk`
- `ldle.d rd, rj, rk`
- `stgt.b rd, rj, rk`
- `stgt.h rd, rj, rk`
- `stgt.w rd, rj, rk`
- `stgt.d rd, rj, rk`
- `stle.b rd, rj, rk`
- `stle.h rd, rj, rk`
- `stle.w rd, rj, rk`
- `stle.d rd, rj, rk`

上述指令和普通访存指令的区别是，在实际访存前，会进行地址判断。
`gt` 即判断 `rj > rk`，`le` 即判断 `rj <= rk`，如果不满足条件，会触发 `SIGSYS` 异常。
特别地，如果同时涉及非对齐访问，且处理器不支持，会直接触发 `SIGBUS` 然后终止。

#### 屏障

- `dbar hint`
- `ibar hint`

`dbar` 是数据屏障，`ibar` 是指令屏障，详细见手册（手册竟然把“屏障”称为“栅障”）。

#### `ll`/`sc`

- `ll.w rd, rj, si14`
- `sc.w rd, rj, si14`
- `ll.d rd, rj, si14`
- `sc.d rd, rj, si14`

功能和 MIPS、ARM 的 `ll`/`sc` 基本一样，区别是成功是 `1`，失败是 `0`。
如果数据不是对齐的，会触发 `SIGBUS` 异常，详细见手册。

#### 原子操作

- `amswap.w rd, rk, rj`
- `amswap.d rd, rk, rj`
- `amadd.w rd, rk, rj`
- `amadd.d rd, rk, rj`
- `amand.w rd, rk, rj`
- `amand.d rd, rk, rj`
- `amor.w rd, rk, rj`
- `amor.d rd, rk, rj`
- `amxor.w rd, rk, rj`
- `amxor.d rd, rk, rj`
- `ammax.w rd, rk, rj`
- `ammax.d rd, rk, rj`
- `ammin.w rd, rk, rj`
- `ammin.d rd, rk, rj`
- `ammax.wu rd, rk, rj`
- `ammax.du rd, rk, rj`
- `ammin.wu rd, rk, rj`
- `ammin.du rd, rk, rj`
- `amswap_db.w rd, rk, rj`
- `amswap_db.d rd, rk, rj`
- `amadd_db.w rd, rk, rj`
- `amadd_db.d rd, rk, rj`
- `amand_db.w rd, rk, rj`
- `amand_db.d rd, rk, rj`
- `amor_db.w rd, rk, rj`
- `amor_db.d rd, rk, rj`
- `amxor_db.w rd, rk, rj`
- `amxor_db.d rd, rk, rj`
- `ammax_db.w rd, rk, rj`
- `ammax_db.d rd, rk, rj`
- `ammin_db.w rd, rk, rj`
- `ammin_db.d rd, rk, rj`
- `ammax_db.wu rd, rk, rj`
- `ammax_db.du rd, rk, rj`
- `ammin_db.wu rd, rk, rj`
- `ammin_db.du rd, rk, rj`

不得不吐槽，这里 `rj` 和 `rk` 是反的。

`rj` 是地址，`rk` 是新值，把 `rj` 处的值取出，和 `rk` 的值进行对应操作，结果放在 `rd` 中。
如果数据不是对齐的，会触发 `SIGBUS` 异常，详细见手册。

#### CRC 校验

- `crc.w.b.w rd, rj, rk`
- `crc.w.h.w rd, rj, rk`
- `crc.w.w.w rd, rj, rk`
- `crc.w.d.w rd, rj, rk`
- `crcc.w.b.w rd, rj, rk`
- `crcc.w.h.w rd, rj, rk`
- `crcc.w.w.w rd, rj, rk`
- `crcc.w.d.w rd, rj, rk`

见手册。

#### 转移

- `beqz rj, offs`
- `bnez rj, offs`
- `jirl rd, rj, offs`
- `b offs`
- `bl offs`
- `beq rj, rd, offs`
- `bne rj, rd, offs`
- `blt rj, rd, offs`
- `bge rj, rd, offs`
- `bltu rj, rd, offs`
- `bgeu rj, rd, offs`

不得不吐槽，这里 `rj` 和 `rd` 也是反的。

"b" 是 "branch" 的缩写，"bl" 是 "branch and linkage" 的缩写，"z" 是 "zero" 的缩写，"eq" 是 "equal" 的缩写，"ne" 是 "not equal" 的缩写，"lt" 是 "less than" 的缩写，"ge" 是 "great equal" 的缩写。
带 "u" 的指无符号比较。

基本看字面意思就能明白这些指令是干嘛的。
`b`/`bl`/`jirl` 都是无条件转移，但 `bl` 会把下一条指令的地址放 `$ra` 里，然后跳转到 `offs` 处；`jirl` 会把下一条指令的地址放 `rd`，然后跳转到 `rj + offs` 处。
特别地，当 `jirl` 的 `rd == rj` 时，是能正确跳转的，不用担心 `rj` 被先覆盖掉了。

需要注意，如果是手写汇编，`offs` 是需要左移两位的，即必须是 4 字节对齐的值。

#### 断言

- `asrtle.d rj, rk`
- `asrtgt.d rj, rk`

"asrt" 应该是 "assert" 的缩写，在不满足比较条件时，上述指令会触发 `SIGSYS` 异常。

#### 计时器

- `rdtimel.w rd, rj`
- `rdtimeh.w rd, rj`
- `rdtime.d rd, rj`

见手册。

#### 陷阱

- `break code`

触发一个异常（手册竟然把“异常”称为“例外”）。

目前我发现的一个用途是在发现除数是 `0` 后用一条 `break` 指令告知内核，因为除法指令本身不会产生异常。

#### 系统调用

- `syscall code`

触发系统调用异常。

目前 LoongArch/Linux 系统调用支持 6 个参数，使用 `syscall 0` 前应该先设置好参数寄存器 `$a0`-`$a5`，并把系统调用号放 `$a7` 中，系统调用的返回值会在 `$a0` 中。

#### 探测

- `cpucfg rd, rj`

用于探测 CPU 的一些功能特性，详细见手册。

测试发现，`rj` 里的值似乎只有低 14 位起作用，不过一般也不会传那么大的数字。

### 基础浮点指令

#### 浮点运算

- `fadd.s fd, fj, fk`
- `fadd.d fd, fj, fk`
- `fsub.s fd, fj, fk`
- `fsub.d fd, fj, fk`
- `fmul.s fd, fj, fk`
- `fmul.d fd, fj, fk`
- `fdiv.s fd, fj, fk`
- `fdiv.d fd, fj, fk`
- `fmadd.s fd, fj, fk, fa`
- `fmadd.d fd, fj, fk, fa`
- `fmsub.s fd, fj, fk, fa`
- `fmsub.d fd, fj, fk, fa`
- `fnmadd.s fd, fj, fk, fa`
- `fnmadd.d fd, fj, fk, fa`
- `fnmsub.s fd, fj, fk, fa`
- `fnmsub.d fd, fj, fk, fa`
- `fmax.s fd, fj, fk`
- `fmax.d fd, fj, fk`
- `fmin.s fd, fj, fk`
- `fmin.d fd, fj, fk`
- `fmaxa.s fd, fj, fk`
- `fmaxa.d fd, fj, fk`
- `fmina.s fd, fj, fk`
- `fmina.d fd, fj, fk`
- `fabs.s fd, fj`
- `fabs.d fd, fj`
- `fneg.s fd, fj`
- `fneg.d fd, fj`
- `fsqrt.s fd, fj`
- `fsqrt.d fd, fj`
- `frecip.s fd, fj`
- `frecip.d fd, fj`
- `frsqrt.s fd, fj`
- `frsqrt.d fd, fj`
- `fscaleb.s fd, fj, fk`
- `fscaleb.d fd, fj, fk`
- `flogb.s fd, fj`
- `flogb.d fd, fj`
- `fcopysign.s fd, fj, fk`
- `fcopysign.d fd, fj, fk`
- `fclass.s fd, fj`
- `fclass.d fd, fj`

以 `fneg` 为例，一元运算符即 `fd = -fj`；
以 `fadd` 为例，二元浮点运算即 `fd = fj + fk`；
以 `fmadd` 为例，三元浮点运算即 `fd = fj * fk + fa`；
以 `fnmadd` 为例，带 "n" 的是在运算结果上再取负，即 `fd = -(fj * fk + fa)`。
以此类推。

大部分都能根据指令名推测出用途，特别说明一下 `fmina`/`fmaxa` 是取绝对值较小/较大的数。
`fcopysign`/`fclass` 等都对应相应的 IEEE 754-2008 中的操作，详细见手册。

#### 浮点比较

- `fcmp.caf.s cd, fj, fk`
- `fcmp.caf.d cd, fj, fk`
- `fcmp.saf.s cd, fj, fk`
- `fcmp.saf.d cd, fj, fk`
- `fcmp.clt.s cd, fj, fk`
- `fcmp.clt.d cd, fj, fk`
- `fcmp.slt.s cd, fj, fk`
- `fcmp.slt.d cd, fj, fk`
- `fcmp.ceq.s cd, fj, fk`
- `fcmp.ceq.d cd, fj, fk`
- `fcmp.seq.s cd, fj, fk`
- `fcmp.seq.d cd, fj, fk`
- `fcmp.cle.s cd, fj, fk`
- `fcmp.cle.d cd, fj, fk`
- `fcmp.sle.s cd, fj, fk`
- `fcmp.sle.d cd, fj, fk`
- `fcmp.cun.s cd, fj, fk`
- `fcmp.cun.d cd, fj, fk`
- `fcmp.sun.s cd, fj, fk`
- `fcmp.sun.d cd, fj, fk`
- `fcmp.cult.s cd, fj, fk`
- `fcmp.cult.d cd, fj, fk`
- `fcmp.sult.s cd, fj, fk`
- `fcmp.sult.d cd, fj, fk`
- `fcmp.cueq.s cd, fj, fk`
- `fcmp.cueq.d cd, fj, fk`
- `fcmp.sueq.s cd, fj, fk`
- `fcmp.sueq.d cd, fj, fk`
- `fcmp.cule.s cd, fj, fk`
- `fcmp.cule.d cd, fj, fk`
- `fcmp.sule.s cd, fj, fk`
- `fcmp.sule.d cd, fj, fk`
- `fcmp.cne.s cd, fj, fk`
- `fcmp.cne.d cd, fj, fk`
- `fcmp.sne.s cd, fj, fk`
- `fcmp.sne.d cd, fj, fk`
- `fcmp.cor.s cd, fj, fk`
- `fcmp.cor.d cd, fj, fk`
- `fcmp.sor.s cd, fj, fk`
- `fcmp.sor.d cd, fj, fk`
- `fcmp.cune.s cd, fj, fk`
- `fcmp.cune.d cd, fj, fk`
- `fcmp.sune.s cd, fj, fk`
- `fcmp.sune.d cd, fj, fk`

手册上归类为 `fcmp.cond.s` 和 `fcmp.cond.d`，详细见手册。

#### 浮点转换

- `fcvt.s.d fd, fj`
- `fcvt.d.s fd, fj`
- `ftintrm.w.s fd, fj`
- `ftintrm.w.d fd, fj`
- `ftintrm.l.s fd, fj`
- `ftintrm.l.d fd, fj`
- `ftintrp.w.s fd, fj`
- `ftintrp.w.d fd, fj`
- `ftintrp.l.s fd, fj`
- `ftintrp.l.d fd, fj`
- `ftintrz.w.s fd, fj`
- `ftintrz.w.d fd, fj`
- `ftintrz.l.s fd, fj`
- `ftintrz.l.d fd, fj`
- `ftintrne.w.s fd, fj`
- `ftintrne.w.d fd, fj`
- `ftintrne.l.s fd, fj`
- `ftintrne.l.d fd, fj`
- `ftint.w.s fd, fj`
- `ftint.w.d fd, fj`
- `ftint.l.s fd, fj`
- `ftint.l.d fd, fj`
- `ffint.s.w fd, fj`
- `ffint.s.l fd, fj`
- `ffint.d.w fd, fj`
- `ffint.d.l fd, fj`
- `frint.s fd, fj`
- `frint.d fd, fj`

`fcvt` 是单双精度浮点数的转换，`ftint` 是浮点数转整数，`ffint` 是整数转浮点数，`frint` 是浮点数舍入到整数。
"rm"、"rp"、"rz" 和 "rne" 是 4 种舍入模式，分别对应 `roundToIntegralTowardNegative`、`roundToIntegralTowardPositive`、`roundToIntegralTowardZero` 和 `roundToIntegralTiesToEven`。

注意所有这些操作都是用的浮点寄存器，没有整数寄存器。

#### 浮点搬运

- `fmov.s fd, fj`
- `fmov.d fd, fj`
- `fsel fd, fj, fk, ca`
- `movgr2fr.w fd, rj`
- `movgr2fr.d fd, rj`
- `movgr2frh.w fd, rj`
- `movfr2gr.s rd, fj`
- `movfr2gr.d rd, fj`
- `movfrh2gr.s rd, fj`
- `movgr2fcsr fcsr, rj`
- `movfcsr2gr rd, fcsr`
- `movfr2cf cd, fj`
- `movcf2fr fd, cj`
- `movgr2cf cd, rj`
- `movcf2gr rd, cj`

根据字面意思就能推断是干嘛的，"gr" 指通用寄存器，"fr" 指浮点寄存器，"fcsr" 指浮点控制状态寄存器，"cf" 指条件标志寄存器。

`fsel` 是条件搬运，即 `fd = (ca == 0) ? fj : fk`；
`movgr2fr.w` 是把 `rj` 的低 32 位搬运到 `fd` 低 32 位，高位未定义，实测目前硬件实现是高位也搬运了，等价于 `movgr2fr.d`；
`movgr2frh.w` 是把 `rj` 的低 32 位搬运到 `fd` 高 32 位，低位不变；
`movcf2fr` 和 `movcf2gr` 会清空高位，这一点手册没提。

#### 浮点访存

- `fld.s fd, rj, si12`
- `fst.s fd, rj, si12`
- `fld.d fd, rj, si12`
- `fst.d fd, rj, si12`
- `fldx.s fd, rj, rk`
- `fldx.d fd, rj, rk`
- `fstx.s fd, rj, rk`
- `fstx.d fd, rj, rk`

这部分指令类似于整数的访存指令。

#### 浮点边界检查访存

- `fldgt.s fd, rj, rk`
- `fldgt.d fd, rj, rk`
- `fldle.s fd, rj, rk`
- `fldle.d fd, rj, rk`
- `fstgt.s fd, rj, rk`
- `fstgt.d fd, rj, rk`
- `fstle.s fd, rj, rk`
- `fstle.d fd, rj, rk`

这部分指令类似于整数的边界检查访存指令。

#### 浮点转移

- `bceqz cj, offs`
- `bcnez cj, offs`

这部分指令类似于整数的条件转移指令，只是比较的寄存器是条件标志寄存器。

## 宏指令

### 空操作

- `nop`

这是唯一一条手册中提到的指令别名，gcc 把它作为宏指令，它等价于 `andi $zero, $zero, 0`。

### 无条件转移

- `jr rd`

等价于 `jirl $zero, rd, 0`，通常用于子程序返回。

### 条件转移

- `bgt rj, rd, lable`
- `bgtu rj, rd, lable`
- `ble rj, rd, lable`
- `bleu rj, rd, lable`
- `bltz rj, rd, lable`
- `bgtz rj, rd, lable`
- `blez rj, rd, lable`
- `bgez rj, rd, lable`

就是字面意思，不再展开介绍。

### 搬运

- `move rd, rj`

等价于 `or rd, rj, $zero`。

### 加载立即数

- `li.w rd, s32`
- `li.w rd, u32`
- `li.d rd, s64`
- `li.d rd, u64`

`li.w` 加载一个 32 位立即数，并符号扩展到 64 位；
`li.d` 加载一个 64 位立即数。

加载立即数一共有三种可能的实现，编译器会自动选择合适的：

```text
# 用一条指令加载
ori dst, $zero, imm[11:0]

# 用两条指令加载
lu12i.w dst, imm[31:12]
ori     dst, dst, imm[11:0]

# 用四条指令加载
lu12i.w dst, imm[31:12]
ori     dst, dst, imm[11:0]
lu32i.d dst, imm[51:32]
lu52i.d dst, dst, imm[63:52]
```

### 加载地址

- `la rd, label + addend`
- `la.global rd, label + addend`
- `la.local rd, label + addend`
- `la.pcrel rd, label + addend`
- `la.got rd, label`
- `la.abs rd, label + addend`

`la` 是 `la.global` 的别名，用于加载全局的符号；
`la.local` 用于加载局部的符号，它通常通过 `pcaddi` 来实现。
平时编程使用 `la.local` 和 `la.global` 即可。

从加载途径划分，加载地址可以细分为 PC 相对偏移加载（`la.pcrel`）、GOT 表加载（`la.got`）和绝对地址加载（`la.abs`）。
但不推荐使用这几个宏，除非你非常清楚自己在做什么，通常交给编译器去自己判断会更好。

加载地址的细节和重定位有关，这里不做讨论。

## ABI 类型

| 名称 | 描述 |
| --- | --- |
| lp64s | 使用 64 位通用寄存器和栈传参，数据模型为 LP64 |
| lp64f | 使用 64 位通用寄存器，32 位浮点寄存器和栈传参，数据模型为 LP64 |
| lp64d | 使用 64 位通用寄存器，64位浮点寄存器和栈传参，数据模型为 LP64 |
| ilp32s | 使用 32 位通用寄存器和栈传参，数据模型为 ILP32 |
| ilp32f | 使用 32 位通用寄存器，32 位浮点寄存器和栈传参，数据模型为 ILP32 |
| ilp32d | 使用 32 位通用寄存器，64 位浮点寄存器和栈传参，数据模型为 ILP32 |

其中，对 C 语言的类型而言，LP64 和 ILP32 数据模型的区别仅在于 `long` 类型和指针类型的长度，前者是 64 位，而后者是 32 位。

## 过程调用约定

以下内容针对 LP64D ABI。

### 参数传递

通用寄存器通常用于传递非浮点数，浮点寄存器通常用于传递浮点数。
在没有可用的浮点寄存器时，浮点数也通过通用寄存器传递；
当通用寄存器也不够用时，再选择栈传参。

特别地，对于 32 位无符号整数（`unsigned int`），如果由通用寄存器来传递，则会被符号扩展到 64 位；
如果由栈来传递，则会被零扩展到 64 位。

#### 标量类型

说来惭愧，标量这个概念我其实是第一次听说，下面摘自[万能的互联网](http://c.biancheng.net/ref/34.html)：

> 标量类型（Scalar type）是相对复合类型（Compound type）来说的：标量类型只能有一个值，而复合类型可以包含多个值。
> 在C语言中，整数类型（int、short、long等）、字符类型（char、wchar_t等）、枚举类型（enum）、小数类型（float、double等）、布尔类型（bool）都属于标量类型，一份标量类型的数据只能包含一个值。
> 结构体（struct）、数组、字符串都属于复合类型，一份复合类型的数据可以包含多个标量类型的值，也可以包含其他复合类型的值。

对小于 64 位的标量，如果是浮点数，那么使用浮点寄存器，反之使用通用寄存器；
如果没有可用的浮点寄存器，那么也使用通用寄存器；
如果没有可用的通用寄存器，那么使用栈传递。

对于类似 `long double` 这种长度为 128 位的数据，优先使用一对（2 个）通用寄存器来传递，且低 64 位存在寄存器编号更低的寄存器里，高 64 位存在寄存器编号更高的寄存器里；
如果只有一个通用寄存器可用，低 64 位存寄存器，高 64 位存栈上；
如果没有通用寄存器可用，则使用栈传递。

#### 复合类型

结构体、联合体、复数、可变参数的情况比较复杂，具体可用参考[文档](https://loongson.github.io/LoongArch-Documentation/LoongArch-ELF-ABI-EN.html)。

### 返回值

- 对于非浮点数，返回值放在 `$a0` 中，如果超过了 64 位，那 `$a1` 也会被使用。
- 同样的，对于浮点数，返回值放 `$fa0` 中，如果超过了 64 位，那 `$a1` 也会被使用。
- 如果超过了两个寄存器的长度，那么将要返回的变量的地址存入 `$a0`。

### 栈

- 栈是向下增长的，栈指针必须 16 字节对齐，且入口处栈指针应该指向传给它的第一个参数（如果有的话）。
- 栈的分布从高到低依次是：参数、保存的返回地址、保存的寄存器、局部变量，当然并不是一定要有的，比如某个函数可以不接受参数，或者不需要保存返回地址、寄存器等。
- 不应对低于栈指针的数据作保证，即应该先对栈指针做减法来分配，然后再使用这段空间。
