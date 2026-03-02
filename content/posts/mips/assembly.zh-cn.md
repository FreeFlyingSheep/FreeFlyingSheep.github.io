---
title: "MIPS 汇编"
date: 2020-07-10
description: ""
menu:
  sidebar:
    name: "MIPS 汇编"
    identifier: mips-assembly
    parent: mips
    weight: 100
tags: ["MIPS", "汇编"]
categories: ["MIPS"]
---

根据 *[Programmed Introduction to MIPS Assembly Language](http://programmedlessons.org/AssemblyTutorial/index.html)* 和《计算机组成与设计——软件/硬件接口》（原书第 5 版）整理。

<!--more-->

## 寄存器

### 通用寄存器

MIPS 提供 32 个 32 位的通用寄存器。

| 寄存器 | 助记符 | 用途 |
| --- | --- | --- |
| `$0` | `zero` | 常数零 |
| `$1` | `$at` | 汇编器保留 |
| `$2`，`$3` | `$v0`，`$v1` | 子程序返回值 |
| `$4`-`$7` | `$a0`-`$a3` | 子程序参数 |
| `$8`-`$15` | `$t0`-`$t7` | 临时寄存器（调用者保存） |
| `$16`-`$23` | `$s0`-`$s7` | 保存寄存器（被调用者保存） |
| `$24`，`$25` | `$t8`，`$t9` | 临时寄存器（调用者保存） |
| `$26`，`$27` | `$k0`，`$k1` | 操作系统保留 |
| `$28` | `$gp` | 全局指针 |
| `$29` | `$sp` | 栈指针 |
| `$30` | `$fp` | 帧指针 |
| `$31` | `$ra` | 返回地址 |

### 特殊寄存器

MIPS 提供一对 32 位寄存器 `hi` 和 `lo` 来帮助完成乘法、除法运算。

### 浮点寄存器

MIPS 有 32 个 32 位的浮点寄存器 `$f0` – `$f31`。

## 指令格式

### R 型（用于寄存器）

| 字段名 | `op` | `rs` | `rt` | `rd` | `shamt` | `funct` |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 含义 | 操作码 | 第一个源操作数寄存器 | 第二个源操作数寄存器 | 目的寄存器 | 位移量 | 功能码 |
| 长度 | 6 | 5 | 5 | 5 | 5 | 6 |

由于 `shamt` 用 5 位表示，所以 0 <= `shamt` < 2^5。

### I 型（用于立即数）

| 字段名 | `op` | `rs` | `rt` | `immediate` |
| :---: | :---: | :---: | :---: | :---: |
| 含义 | 操作码 | 源操作数寄存器 | 目的寄存器 | 立即数 |
| 长度 | 6 | 5 | 5 | 16 |

由于 `immediate` 用 16 位表示，所以 -2^15 <= `immediate` < 2^15 。

注意，此处 `rt` 的含义与上面不同，且后面的指令如不注明，`immediate` 默认被**符号扩展**为 32 位！

### J 型（用于分支和跳转）

| 字段名 | `op` | `address` |
| :---: | :---: | :---: |
| 含义 | 操作码 | 地址 |
| 长度 | 6 | 26 |

特别的，对于条件分支指令，地址字段可能又被划分为了多个部分。

## 基本汇编指令

### 算术运算指令

#### 加法指令

| 指令格式 | 含义 |
| --- | --- |
| `addu rd,rs,rt` | `rd = rs + rt`（溢出时不产生异常） |
| `add rd,rs,rt` | `rd = rs + rt`（溢出时产生异常） |
| `addiu rt,rs,immediate` | `rt = rs + immediate`（溢出时不产生异常） |
| `addi rt,rs,immediate` | `rt = rs + immediate`（溢出时产生异常） |

注意，MIPS 只有 32 位算术运算，在高级语言中需要进行 16 位或 8 位算术运算时，编译器可能产生更多的指令来模拟这些运算，这意味这 32 位算术运算的速度往往会快于其他情况，而不是数据长度越短运算越快。

#### 减法指令

| 指令格式 | 含义 |
| --- | --- |
| `subu rd,rs,rt` | `d = rs-rt`（溢出时不产生异常） |
| `sub rd,rs,t` | `d = rs-rt`（溢出时产生异常） |
| `subiu rd,rs,immediate` | `d = rs-immediate` （溢出时不产生异常） |
| `subi rd,rs,immediate` | `d = rs-immediate`（溢出时产生异常） |

#### 乘法指令

| 指令格式 | 含义 |
| --- | --- |
| `mult rd,rt` | `hilo = rd * rt`（有符号数） |
| `multu rd,rt` | `hilo = rd * rt`（无符号数） |

- `hi`：结果的高 32 位
- `lo`：结果的低 32 位

注意，此处指令中带的 `u` 和加减法指令含义不同。对于**加减法**，有符号和无符号数的运算相同，区别仅在**带 `u` 的版本溢出时不产生异常**；而对于**乘除法**，有符号和无符号数的运算不同，且无论是否溢出，都不产生异常，区别仅在**带 `u` 的版本特指无符号数运算**。

#### 除法指令

| 指令格式 | 含义 |
| --- | --- |
| `div rd,rt` | `lo = rd / rt; hi = rd % rt`（有符号数） |
| `divu rd,rt` | `lo = rd / rt; hi = rd % rt`（无符号数） |

- `lo`：商
- `hi`：余数

### 逻辑运算指令

#### 位运算指令

| 指令格式 | 含义 |
| --- | --- |
| `ori rt,rs,immediate` | `rt = rs | immediate` |
| `andi rt,rs,immediate` | `rt = rs & immediate` |
| `xori d,s,immediate` | `rt = rs ^ immediate` |
| `or rd,rs,rt` | `rd = rs | rt` |
| `and rd,rs,rt` | `rd = rs & rt` |
| `xor rd,rs,rt` | `rd = rs ^ rt` |
| `nor rd,rs,rt` | `rd = ~(rs | rt)` |

#### 移位指令

| 指令格式 | 含义 |
| --- | --- |
| `sll rd,rt,shamt` | `rd = rt << shamt` |
| `srl rd,rt,shamt` | `rd = rt >> shamt`（逻辑右移） |
| `sra rd,rt,shamt` | `rd = rt >> shamt`（算术右移） |

#### 特殊用法

| 指令格式 | 含义 |
| --- | --- |
| `nor rd,rs,$0` | `rd = ~rs` |
| `or rd,rs,$0` | `rd = rs` |
| `sll $0,$0,0` | 空操作 |

由于对 `$0` 的改写无实际意义，所以可以借助 `$0` 实现空操作。

### 数据传输指令

| 指令格式 | 含义 |
| --- | --- |
| `lw rt,immediate(rs)` | `rt = memory[rs + immediate]` |
| `sw rt,immediate(rs)` | `memory[rs + immediate] = rt` |
| `lh rt,immediate(rs)` | `rt = memory[rs + immediate]` |
| `lhu rt,immediate(rs)` | `rt = memory[rs + immediate]`（零扩展） |
| `sh rt,immediate(rs)` | `memory[rs + immediate] = rt` |
| `lb rt,immediate(rs)` | `rt = memory[rs + immediate]` |
| `lbu rt,immediate(rs)` | `rt = memory[rs + immediate]`（零扩展） |
| `sb rt,immediate(rs)` | `memory[rs + immediate] = rt` |
| `lui rt,immediate` | `rt = immediate * 2^16` |

- `w`：字，32 位
- `h`：半字，16位
- `b`：字节，8 位

MIPS 可以使用大端序和小端序，具体由系统设计人员决定。

若要生成 32 位立即数用于寄存器基址寻址，可以先用 `lui` 指令加载高位，再用 `ori` 或者 `lw` 指令填充低位，以完成 `$12 = memory[$13 + 0xC]` 为例，有以下两种方式：

    # 方式一
    lui $13,0x0060
    ori $13,0x5000 # 简化版 ori 指令，同 ori $13,$13,0x5000
    lw  $12,0xC($13)

    # 方式二
    lui $13,0x0060
    lw  $12,0x500C($13)

### 无条件跳转指令

| 指令格式 | 含义 |
| --- | --- |
| `j address` | `goto address * 4` |
| `jal address` | `$ra = PC + 4; goto address * 4` |
| `jr register` | `goto register` |

MIPS 的指令总是 32 位的，因此指令是 4 字节对齐的，32 位地址的低 2 位永远为 0。将 32 位目标地址右移 2 位，然后存储低 26 位，这 26 位的地址实际可以替代 `PC` 的低 28 位。

注意，使用 `j` 指令时，`PC` 的高 4 位是不变的，这意味着寻址界限为 256 MB，超过该界限时，应该使用寄存器跳转指令 （通常编译器会完成这些操作）。

可以使用符号地址来简化编程：

    main: 
      ...
      j main

### 条件分支指令

#### 条件跳转指令

| 指令格式 | 含义 |
| --- | --- |
| `beq register1,register2,address` | `if (register1 == register2) goto PC + 4 + address * 4` |
| `bne register1,register2,address` | `if (register1 != register2) goto PC + 4 + address * 4` |
| `bltz register,label` | `if (register < 0) goto PC + 4 + address * 4` |
| `bgez register,label` | `if (register >= 0) goto PC + 4 + address * 4` |

条件跳转指令使用 PC 相对寻址。

#### 条件置位指令

| 指令格式 | 含义 |
| --- | --- |
| `slt rd,rs,rt` | `if (rs < rt) rd = 1; else rd = 0`（有符号数） |
| `sltu rd,rs,rt` | `if (rs < rt) rd = 1; else rd = 0`（无符号数） |
| `slti rt,rs,immediate` | `if (rs < immediate) rt = 1; else rt = 0`（有符号数） |
| `sltiu rt,rs,immediate` | `if (rs < immediate) rt = 1; else rt = 0`（无符号数） |

再次强调，虽然 `sltiu` 代表无符号版本，但立即数会先被**符号展开**，然后再作为无符号数比较。

### 其他指令

| 指令格式 | 含义 |
| --- | --- |
| `mfhi rd` | `rd = hi` |
| `mflo rd` | `rd = lo` |
| `syscall` | 系统调用 |

注意，在早期的 MIPS 处理器上，`mfhi` 和 `mflo` 指令后的两条指令，不得出现乘法和除法指令，其原因涉及 MIPS 流水线的工作方式。

## 伪指令

### 算术运算伪指令

| 指令格式 | 含义 |
| --- | --- |
| `addu d,s,x` | `d = s + x` |
| `subu d,s,x` | `d = s-x` |
| `negu d,s` | `d = -s` |
| `mul d,s,t` | `d = s * t`（结果不超过 32 位） |
| `div d,s,t` | `d = s / t`（有符号数） |
| `divu d,s,t` | `d = s / t`（无符号数） |
| `remu d,s,t` | `d = s % t`（无符号数） |
| `abs d,s` | `d = |s|` |

其中，`x` 可以是寄存器、 16 位立即数或 32 位立即数，后面的 `x` 含义相同，不再重复。

### 逻辑运算伪指令

| 指令格式 | 含义 |
| --- | --- |
| `not d,s` | `d = ~s` |
| `or d,s,x` | `d = s | x` |
| `and d,s,x` | `d = s & x` |
| `rol d,s,t` | `d = s << t`（循环左移） |
| `ror d,s,t` | `d = s >> t`（循环右移） |

### 数据传输伪指令

| 指令格式 | 含义 |
| --- | --- |
| `move d,s` | `d = s` |
| `li d,value` | `d = value`（`value` 可以是 16 位或 32 位） |
| `la d,exp` | `d = address(exp)`（`exp` 通常是一个符号地址） |
| `lw d,exp` | `d = memory[exp]` |
| `sw d,exp` | `memory[exp] = d` |

伪指令还允许借助符号地址来寻址，示例如下：

    li $t1,2                 # index 2
    lb $v0,data($t1)         # $v0 = data[$t1]
    ...
      
    data: .byte  6,34,12,-32, 90

### 条件分支伪指令

#### 条件跳转伪指令

| 指令格式 | 含义 |
| --- | --- |
| `b label` | `goto label` |
| `beq s,x,label` | `if (s == x) goto label` |
| `beqz s,label` | `if (s == 0) goto label` |
| `bge s,x,label` | `if (s >= x) goto label`  (有符号数) |
| `bgeu s,x,label` | `if (s >= x) goto label`  (无符号数) |
| `bgez s,label` | `if (s >= 0) goto label`  (有符号数) |
| `bgt s,x,label` | `if (s > x) goto label`  (有符号数) |
| `bgtu s,x,label` | `if (s > x) goto label`  (无符号数) |
| `bgtz s,label` | `if (s > 0) goto label`  (有符号数) |
| `ble s,x,label` | `if (s <= x) goto label`  (有符号数) |
| `bleu s,x,label` | `if (s <= x) goto label`  (无符号数) |
| `blez s,label` | `if (s <= 0) goto label`  (有符号数) |
| `blt s,x,label` | `if (s < x) goto label`  (有符号数) |
| `bltu s,x,label` | `if (s < x) goto label`  (无符号数) |
| `bltz s,label` | `if (s < 0) goto label`  (有符号数) |
| `bnez s,label` | `if (s != 0) goto label` |  
| `bne s,x,label` | `if (s != x) goto label` |

注意，所有的分支指令都只能跳转到分支附近的位置，它只使用 16 位来表示地址。

#### 条件置位伪指令

| 指令格式 | 含义 |
| --- | --- |
| `slt d,s,x` | `if (s < x) d = 1; else d = 0` |
| `seq d,s,x` | `if (s == x) d = 1; else d = 0` |  
| `sge d,s,x` | `if (s >= x) d = 1; else d = 0` (有符号数) |
| `sgeu d,s,x` | `if (s >= x) d = 1; else d = 0` (无符号数) |
| `sgt d,s,x` | `if (s > x) d = 1; else d = 0` (有符号数) |
| `sgtu d,s,x` | `if (s > x) d = 1; else d = 0` (无符号数) |
| `sle d,s,x` | `if (s <= x) d = 1; else d = 0` (有符号数) |
| `sleu d,s,x` | `if (s <= x) d = 1; else d = 0` (无符号数) |
| `slt d,s,x` | `if (s < x) d = 1; else d = 0` (有符号数) |
| `slti d,s,immediate` | `if (s < immediate) d = 1; else d = 0` (有符号数) |
| `sltu d,s,x` | `if (s < x) d = 1; else d = 0` (无符号数) |
| `sltiu d,s,immediate` | `if (s < immediate) d = 1; else d = 0` (无符号数) |
| `sne d,s,x` | `if (s != x)` |

### 浮点伪指令

| 指令格式 | 含义 |
| --- | --- |
| `l.s fd,address` | `fd = memory[address]` |
| `s.s fd,address` | `memory[address] = fd` |
| `li.s fd,value` | `fd = value` |
| `abs.s fd,fs` | `$d = |fs|` |
| `add.s fd,fs,ft` | `fd = fs + ft` |
| `sub.s fd,fs,ft` | `fd = fs-ft` |
| `mul.s fd,fs,ft` | `fd = fs * ft` |
| `div.s fd,fs,ft` | `fd = fs / ft` |
| `neg.s fd,fs` | `fd = -fs` |
| `mov.s fd, fs` | `fd = fs` |
| `mtc1 rs, fd` | `fd = rs`（直接赋值，不进行转换，注意此处目标寄存器和源寄存器顺序） |
| `mfc1 rd, fs` | `rd = fs`（直接赋值，不进行转换） |
| `c.eq.s fs, ft` | `if (fs == ft) condition bit = 1; else condition bit = 0` |
| `c.lt.s fs, ft` | `if (fs < ft) condition bit = 1; else condition bit = 0` |
| `c.le.s fs, ft` | `if (fs <= ft) condition bit = 1; else condition bit = 0` |
| `bc1t label` | `if (condition bit = 1) goto label` |
| `bc1f label` | `if (condition bit = 0) goto label` |

部分 MIPS 处理器只允许在单精度浮点指令中使用 `$f0`、`$f2` 、 ... 、 `$f30` 寄存器。以上 `.s` 结尾的都是单精度浮点指令，对于双精度，将 `s` 替换为 `d` 即可，这样 MIPS 将使用寄存器对来进行运算，寄存器对的名称是 `$f0`、`$f2` 、 ... 、 `$f30`。

### 其他伪指令

| 指令格式 | 含义 |
| --- | --- |
| `nop` | 空操作 |

## 汇编器指令

| 语法 | 用途 |
| --- | --- |
| `.text` | 表明代码段的开始 |
| `.globl` | 表明标识符是一个全局符号 |
| `.data` | 表明数据段的开始 |
| `.byte` | 放置一个 8 位的整数，多个整数用逗号隔开 |
| `.word` | 放置一个 32 位的整数，多个整数用逗号隔开 |
| `.space` | 在内存中保留的字节数 |
| `.ascii` | 放置一个 ASCII 字符串 |
| `.asciiz` | 放置一个 C 风格的字符串 (以 `\0` 结尾的 ASCII 字符串) |
| `.float` | 放置一个 32 位的单精度浮点数，多个浮点数用逗号隔开 |

## 栈

在 MIPS 中，栈是向下增长的。

通过以下代码实现 `push` 操作：

    # PUSH the item in $t0:
    subu $sp,$sp,4      #   point to the place for the new item,
    sw   $t0,($sp)      #   store the contents of $t0 as the new top.

通过以下代码实现 `pop` 操作：

    # POP the item into $t0:
    lw   $t0,($sp)      #   Copy top item to $t0.
    addu $sp,$sp,4      #   Point to the item beneath the old top.

## 子程序连接

### 基于堆栈的连接约定

1. 调用子程序（由调用者完成）：
    1.1. 将需要保存的 `$t0`-`$t9` 寄存器入栈，子程序可能会修改它们。
    1.2. 将参数放入 `$a0`-`$a3`。
    1.3. 使用 `jal` 指令调用子程序。
2. 子程序 prolog（由子程序在开头完成）：
    2.1. 如果该子程序需要调用其他子程序，将 `$ra` 入栈。
    2.2. 将需要修改的 `$s0`-`$s7` 寄存器入栈。
3. 子程序主体:
    3.1. 子程序可以修改 `$t0`-`$t9`、`$a0`-`$a3` 和 2.2 中保存的寄存器。
    3.2. 如果子程序需要调用其他子程序，也必须遵守这些约定。
4. 子程序 epilog（由子程序在返回前完成）：
    4.1. 将返回值放入 `$v0`-`$v1`。
    4.2. 将 2.2 中保存的寄存器按与入栈时相反的顺序出栈。
    4.3. 如果 `$ra` 在 2.1 中被保存，将它出栈。
    4.4. 使用 `ja $ra` 指令返回。
5. 从子程序重新获得控制权（由调用者完成）：
    5.1. 将 1.1 中保存的寄存器按与入栈时相反的顺序出栈。

### 基于栈帧的连接约定

以下约定假设栈只存储 32 位的数据。

1. 调用子程序（由调用者完成）：
    1.1. 按数字顺序将需要保存的 `$t0`-`$t9` 寄存器入栈。
    1.2. 将参数放入 `$a0`-`$a3`。
    1.3. 使用 `jal` 指令调用子程序。
2. 子程序 prolog（由子程序在开头完成）：
    2.1. 将 `$ra` 入栈。
    2.2. 将调用者的帧指针 `$fp` 入栈。
    2.3. 将需要修改的 `$s0`-`$s7` 寄存器入栈。
    2.4. 初始化帧指针： `$fp = $sp-space`，其中 `space` 指变量占用空间，此处是变量个数的 4 倍。
    2.5. 初始化栈指针： `$sp = $fp`。
3. 子程序主体：
    3.1. 子程序可以修改 `$t0`-`$t9`、`$a0`-`$a3` 和 2.3 中保存的寄存器。
    3.2. 子程序根据 `disp($fp)` 的形式寻址来使用局部变量。
    3.3. 子程序可以通过修改 `$sp` 在栈上存储数据。
    3.4. 如果子程序需要调用其他子程序，也必须遵守这些约定。
4. 子程序 epilog（由子程序在返回前完成）：
    4.1. 将返回值放入 `$v0`-`$v1`。
    4.2. `$sp = $fp + space`。
    4.3. 将 2.3 中保存的寄存器按与入栈时相反的顺序出栈。
    4.4. 将调用者的帧指针 `$fp` 出栈。
    4.5. 将 `$ra` 出栈。
    4.6. 使用 `ja $ra` 指令返回。
5. 从子程序重新获得控制权（由调用者完成）：
    5.1. 将 1.1 中保存的寄存器按与入栈时相反的顺序出栈。

进入子程序时栈结构如下图所示：

![进入子程序时栈的结构](/images/mips/stack.png)
