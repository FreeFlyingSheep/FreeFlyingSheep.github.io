---
title: "Linux/MIPS 启动"
date: 2020-07-17
description: ""
menu:
  sidebar:
    name: "Linux/MIPS 启动"
    identifier: tips-mips-boot
    parent: tips
    weight: 100
tags: ["Linux 内核", "MIPS"]
categories: ["Tips"]
---

阅读 MIPS 启动部分的代码，从内核入口 `kernel_entry` 到 `start_kernel` 的第一个子函数 `lock_kernel`，很多细节我并没理解，所以不进行展开。基于 Linux kernel release 2.6.11.12。由于 Markdown 代码块语法高亮不支持汇编，此处统一用 `C` 标注。

<!--more-->

## 函数

### `kernel_entry`

`arch/mips/kernel/head.S`：

```c
NESTED(kernel_entry, 16, sp) # kernel entry point
    setup_c0_status_pri

#ifdef CONFIG_SGI_IP27
    GET_NASID_ASM t1
    move t2, t1 # text and data are here // t2 = t1
    MAPPED_KERNEL_SETUP_TLB
#endif /* IP27 */

    ARC64_TWIDDLE_PC

    PTR_LA t0, __bss_start # clear .bss // t0 = __bss_start
    LONG_S zero, (t0) // *(t0) = 0
    PTR_LA t1, __bss_stop - LONGSIZE // t1 = __bss_stop - LONGSIZE
1:
    PTR_ADDIU t0, LONGSIZE // t0 += LONGSIZE
    LONG_S zero, (t0) // *(t0) = 0
    bne t0, t1, 1b // if (t0 != t1) goto 1b

    LONG_S a0, fw_arg0 # firmware arguments
    LONG_S a1, fw_arg1
    LONG_S a2, fw_arg2
    LONG_S a3, fw_arg3

    PTR_LA $28, init_thread_union // gp = init_thread_union
    PTR_ADDIU sp, $28, _THREAD_SIZE - 32 // sp = gp + _THREAD_SIZE - 32
    set_saved_sp sp, t0, t1
    PTR_SUBU sp, 4 * SZREG # init stack pointer // sp = 4 * SZREG

    j start_kernel
    END(kernel_entry)
```

`init_thread_union` 定义于 `arch/mips/kernel/init_task.c`。

### `start_kernel`

`init/main.c`：

```c
/*
 * Activate the first processor.
 */

asmlinkage void __init start_kernel(void)
{
    char * command_line;
    extern struct kernel_param __start___param[], __stop___param[];
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
    lock_kernel();
    ...
```

### `lock_kernel`

`include/linux/smp_lock.h`：

```c
#ifdef CONFIG_LOCK_KERNEL

extern void __lockfunc lock_kernel(void) __acquires(kernel_lock);
extern void __lockfunc unlock_kernel(void) __releases(kernel_lock);

#else

#define lock_kernel() do { } while(0)
#define unlock_kernel() do { } while(0)

#endif /* CONFIG_LOCK_KERNEL */
```

`lock_kernel` 和 `unlock_kernel` 的具体实现在 `lib/kernel_lock.c` 中。

## 汇编器指令

主要参考文档 *[Using as](https://sourceware.org/binutils/docs-2.34/as/index.html)*。

### `.set noreorder`

设置汇编器不移动指令来填充延迟槽。

### `.set push` 和 `.set pop`

见 [Directives to save and restore options](https://sourceware.org/binutils/docs-2.34/as/MIPS-Option-Stack.html#index-_002eset-push)。

可以先用 `.set push` 保存汇编器配置，然后执行一些修改 (比如配置 `.set noreorder`)，最后用 `.set pop` 恢复之前的汇编器配置。

### `.comm`

见 [`.comm symbol , length`](https://sourceware.org/binutils/docs-2.34/as/Comm.html#Comm)。

在大部分情况下，可以简单理解为定义一个变量，变量名为 `symbol` ，占 `length` 字节，可选的第三个参数代表对齐字节数。

### `.align [abs-expr[, abs-expr[, abs-expr]]]`

见 [`.align [abs-expr[, abs-expr[, abs-expr]]]`](https://sourceware.org/binutils/docs-2.34/as/Align.html#Align)。

通常只用到第一个表达式，如 `.align 8` 代表 8 字节对齐。

### `.type`

见 [`.type`](https://sourceware.org/binutils/docs-2.34/as/Type.html#Type)。

在内核中，基本只用到了 `.type <name>,@<type>` 的形式，用于表明函数类型。

### `.globl`

见 [`.globl`](https://sourceware.org/binutils/docs-2.34/as/Global.html#Global)。

声明变量 (供链接器使用)。

### `.ent`

声明变量 (供调试器使用)。

### `.frame`

`.frame sp, size, ra`，提供给调试器关于栈帧的信息，`sp` 代表指向栈帧的地址、寄存器，`size` 代表栈帧的大小，`ra` 代表保存返回地址的寄存器。

### `.section`

见 [`.section name`](https://sourceware.org/binutils/docs-2.34/as/Section.html#Section)。

### `.previous`

见 [`.previous`](https://sourceware.org/binutils/docs-2.34/as/Previous.html#Previous)。

### `.macro` 和 `.endm`

见 [`.macro`](https://sourceware.org/binutils/docs-2.34/as/Macro.html#Macro)。

## 宏

### `EXPORT`

`include/asm-mips/asm.h`：

```c
/*
* EXPORT - export definition of symbol
*/
#define EXPORT(symbol)                  \
        .globl  symbol;                         \
symbol:
```

声明一个全局变量。

### `__INIT` 和 `__FINIT`

`include/linux/init.h`：

```c
#define __INIT .section ".init.text","ax"
#define __FINIT .previous
```

`.section` 和 `.previous` 见汇编器指令章节的相关内容。

### 32 位和 64 位的兼容

32 位和 64 位情况下的数据长度和指令不同，具体见 `include/asm-mips/asm.h`，该文件中的部分宏定义如下：

```c
/*
 * Size of a register
 */
#ifdef __mips64
#define SZREG 8
#else
#define SZREG 4
#endif

/*
 * How to add/sub/load/store/shift C long variables.
 */
#if (_MIPS_SZLONG == 32)
#define LONG_S sw
#endif

#if (_MIPS_SZLONG == 64)
#define LONG_S sd
#endif

/*
 * How to add/sub/load/store/shift pointers.
 */
#if (_MIPS_SZPTR == 32)
#define PTR_ADDIU addiu
#define PTR_SUBU subu
#define PTR_LA la
#endif

#if (_MIPS_SZPTR == 64)
#define PTR_ADDU daddu
#define PTR_SUBU dsubu
#define PTR_LA dla
#endif
```

### `NESTED`

`include/asm-mips/asm.h`：

```c
/*
* NESTED - declare nested routine entry point
*/
#define NESTED(symbol, framesize, rpc)                  \
        .globl  symbol;                         \
        .align  2;                              \
        .type   symbol,@function;               \
        .ent    symbol,0;                       \
symbol:     .frame  sp, framesize, rpc
```

定义一个函数的头部。

### `setup_c0_status_pri`

`arch/mips/kernel/head.S`：

```c
    /*
     * For the moment disable interrupts, mark the kernel mode and
     * set ST0_KX so that the CPU does not spit fire when using
     * 64-bit addresses.  A full initialization of the CPU's status
     * register is done later in per_cpu_trap_init().
     */
    .macro setup_c0_status set clr
    .set push
    mfc0 t0, CP0_STATUS // t0 = $12
    or t0, ST0_CU0|\set|0x1f|\clr // t0 |= ST0_CU0 | set | 0x1f | clr
    xor t0, 0x1f|\clr // t0 ^= 0x1f | clr
    mtc0 t0, CP0_STATUS // $12 = t0
    .set noreorder
    sll zero,3 # ehb
    .set pop
    .endm

    .macro setup_c0_status_pri
#ifdef CONFIG_MIPS64
    setup_c0_status ST0_KX 0 // setup_c0_status 0x00000080 0，见宏章节的 CP0 寄存器
#else
    setup_c0_status 0 0 // setup_c0_status 0 0
#endif
    .endm
```

如果配置了 `CONFIG_MIPS64`，`setup_c0_status` 会把 CP0 的 `$12` 设置为 `0x80`；反之则设置为 `0`。具体作用注释已经给出，建议参考 *[The MIPS64 and microMIPS64 Privileged Resource Architecture v6.03](https://s3-eu-west-1.amazonaws.com/downloads-mips/documents/MD00091-2B-MIPS64PRA-AFP-06.03.pdf)* 的 `Table 9.49`。

### CP0 寄存器

`include/asm-mips/mipsregs.h`：

```c
/*
 * Coprocessor 0 register names
 */
#define CP0_STATUS $12

/*
 * Bitfields in the R4xx0 cp0 status register
 */
#define ST0_KX 0x00000080
```

### `GET_NASID_ASM`

`arch/mips/kernel/head.S`：

```c
#ifdef CONFIG_SGI_IP27
    /*
     * outputs the local nasid into res.  IP27 stuff.
     */
    .macro GET_NASID_ASM res
    dli \res, LOCAL_HUB_ADDR(NI_STATUS_REV_ID)
    ld \res, (\res)
    and \res, NSRI_NODEID_MASK
    dsrl \res, NSRI_NODEID_SHFT
    .endm
#endif /* CONFIG_SGI_IP27 */
```

### `MAPPED_KERNEL_SETUP_TLB`

定义于 `arch/mips/kernel/head.S`。

### `ARC64_TWIDDLE_PC`

`arch/mips/kernel/head.S`：

```c
    .macro ARC64_TWIDDLE_PC
#if defined(CONFIG_ARC64) || defined(CONFIG_MAPPED_KERNEL)
    /* We get launched at a XKPHYS address but the kernel is linked to
       run at a KSEG0 address, so jump there.  */
    PTR_LA t0, \@f
    jr t0
\@:
#endif
    .endm
```

### `_THREAD_SIZE`

定义于 `arch/mips/kernel/offset.c` ，该文件会根据 `arch/mips/Makefile` 中的规则，生成 `asm/offset.h`。

``` Makefile
# Generate <asm/offset.h>
#
# The default rule is suffering from funny problems on MIPS so we using our
# own ...
#
# ---------------------------------------------------------------------------

define filechk_gen-asm-offset.h
    (set -e; \
     echo "#ifndef _ASM_OFFSET_H"; \
     echo "#define _ASM_OFFSET_H"; \
     echo "/*"; \
     echo " * DO NOT MODIFY."; \
     echo " *"; \
     echo " * This file was generated by arch/$(ARCH)/Makefile"; \
     echo " *"; \
     echo " */"; \
     echo ""; \
     sed -ne "/^@@@/s///p"; \
     echo "#endif /* _ASM_OFFSET_H */" )
endef

prepare: include/asm-$(ARCH)/offset.h

arch/$(ARCH)/kernel/offset.s: include/asm include/linux/version.h \
                   include/config/MARKER

include/asm-$(ARCH)/offset.h: arch/$(ARCH)/kernel/offset.s
    $(call filechk,gen-asm-offset.h)
```

`Makefile`：

``` Makefile
# filechk is used to check if the content of a generated file is updated.
# Sample usage:
# define filechk_sample
#   echo $KERNELRELEASE
# endef
# version.h : Makefile
#   $(call filechk,sample)
# The rule defined shall write to stdout the content of the new file.
# The existing file will be compared with the new one.
# - If no file exist it is created
# - If the content differ the new file is used
# - If they are equal no change, and no timestamp update

define filechk
    @set -e; \
    echo '  CHK     $@'; \
    mkdir -p $(dir $@); \
    $(filechk_$(1)) < $< > $@.tmp; \
    if [ -r $@ ] && cmp -s $@ $@.tmp; then \
        rm -f $@.tmp; \
    else \
        echo '  UPD     $@'; \
        mv -f $@.tmp $@; \
    fi
endef
```

查看文件中 `_THREAD_SIZE` 相关代码，最终该宏相当于 `#define _THREAD_SIZE THREAD_SIZE`。

### `THREAD_SIZE`

`include/asm-mips/thread_info.h`：

```c
/* thread information allocation */
#if defined(CONFIG_PAGE_SIZE_4KB) && defined(CONFIG_MIPS32)
#define THREAD_SIZE_ORDER (1)
#endif
#if defined(CONFIG_PAGE_SIZE_4KB) && defined(CONFIG_MIPS64)
#define THREAD_SIZE_ORDER (2)
#endif
#ifdef CONFIG_PAGE_SIZE_8KB
#define THREAD_SIZE_ORDER (1)
#endif
#ifdef CONFIG_PAGE_SIZE_16KB
#define THREAD_SIZE_ORDER (0)
#endif
#ifdef CONFIG_PAGE_SIZE_64KB
#define THREAD_SIZE_ORDER (0)
#endif

#define THREAD_SIZE (PAGE_SIZE << THREAD_SIZE_ORDER)
```

include/asm-mips/page.h：

```c
/*
 * PAGE_SHIFT determines the page size
 */
#ifdef CONFIG_PAGE_SIZE_4KB
#define PAGE_SHIFT 12
#endif
#ifdef CONFIG_PAGE_SIZE_8KB
#define PAGE_SHIFT 13
#endif
#ifdef CONFIG_PAGE_SIZE_16KB
#define PAGE_SHIFT 14
#endif
#ifdef CONFIG_PAGE_SIZE_64KB
#define PAGE_SHIFT 16
#endif
#define PAGE_SIZE (1UL << PAGE_SHIFT)
```

### `set_saved_sp`

定义于 `include/asm-mips/stackframe.h`，见后续笔记。

### `END`

```c
/*
* END - mark end of function
*/
#define END(function)                                   \
        .end    function;               \
        .size   function,.-function
```

定义一个函数的尾部。

### `CP0_STATUS`

`include/asm-mips/mipsregs.h`：

```c
#define CP0_STATUS $12
```

### `CONFIG_SGI_IP27`

arch/mips/Makefile:

``` Makefile
#
# SGI-IP27 (Origin200/2000)
#
# Set the load address to >= 0xc000000000300000 if you want to leave space for
# symmon, 0xc00000000001c000 for production kernels.  Note that the value must
# be 16kb aligned or the handling of the current variable will break.
#
```

### `asmlinkage`

`include/linux/linkage.h`：

```c
#include <asm/linkage.h>

#ifdef __cplusplus
#define CPP_ASMLINKAGE extern "C"
#else
#define CPP_ASMLINKAGE
#endif

#ifndef asmlinkage
#define asmlinkage CPP_ASMLINKAGE
#endif
```

`include/asm-mips/linkage.h`：

```c
/* Nothing to see here... */
```

在 MIPS 下，`asmlinkage` 只是为了兼容 `C++`。

### `__init`

`include/linux/init.h`：

```c
#define __init __attribute__ ((__section__ (".init.text")))
```

把函数放到 `.init.text` 段。

### `__acquires` 和 `__releases`

`include/linux/compiler.h`：

```c
# define __acquires(x) __attribute__((context(0,1)))
# define __releases(x) __attribute__((context(1,0)))
```

给 Sparse 做代码静态检查，用于保证 `__acquires` 和 `__releases` 成对使用。Sparse 的相关内容可以参考 *[The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/dev-tools/sparse.html)*。

### `__lockfunc`

`include/linux/spinlock.h`：

```c
#define __lockfunc fastcall __attribute__((section(".spinlock.text")))
```

`include/linux/linkage.h`：

```c
#ifndef FASTCALL
#define FASTCALL(x) x
#define fastcall
#endif
```

`include/asm-mips/linkage.h`：

```c
/* Nothing to see here... */
```

在 MIPS 下，`fastcall` 为空，`__lockfunc` 只是为了把函数放到 `.spinlock.text` 段。
