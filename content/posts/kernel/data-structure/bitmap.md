---
title: "内核位图"
date: 2020-10-28
description: ""
menu:
  sidebar:
    name: "内核位图"
    identifier: kernel-data-structure-bitmap
    parent: kernel-data-structure
    weight: 100
tags: ["Linux 内核", "内核数据结构", "位图"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，GCC 扩展语法和内核数据结构部分，简单介绍 Linux 内核位图。

<!--more-->

## 内核位图的使用

下面只列举常用的内核位图函数/宏，头文件为 `include/linux/bitops.h`。

### 创建位图

```c
DECLARE_BITMAP(name,bits);
```

定义一个名称为 `name` 且至少包含 `bits` 位的位图。

### 设置位

```c
void set_bit(unsigned int nr, volatile unsigned long *addr);
```

### 清除位

```c
void clear_bit(int nr, volatile unsigned long *addr);
```

### 改变位

```c
void change_bit(int nr, volatile unsigned long *addr);
```

### 测试并设置位

```c
int test_and_set_bit(int nr, volatile unsigned long *addr);
```

若被测试位为 `0`，返回 `0`，反之返回非 `0` 值。

### 测试并清除位

```c
int test_and_clear_bit(int nr, volatile unsigned long *addr);
```

若被测试位为 `0`，返回 `0`，反之返回非 `0` 值。

### 测试并改变位

```c
int test_and_change_bit(int nr, volatile unsigned long *addr);
```

若被测试位为 `0`，返回 `0`，反之返回非 `0` 值。

### 测试位

```c
int test_bit(int nr, const volatile unsigned long *addr);
```

若被测试位为 `0`，返回 `0`，反之返回非 `0` 值。

## 内核位图的实现

定义内核位图的宏位于 `include/linux/types.h`。内核位操作的实现分为两部分，体系结构无关的实现位于 `include/linux/bitops.h`，x86 体系结构相关的部分位于 `arch/x86/include/asm/bitops.h`。

### 创建位图的实现

`include/linux/types.h`：

```c
#define DECLARE_BITMAP(name,bits) \
        unsigned long name[BITS_TO_LONGS(bits)]
```

`include/linux/bitops.h`：

```c
#define BITS_PER_BYTE     8
#define BITS_TO_LONGS(nr) DIV_ROUND_UP(nr, BITS_PER_BYTE * sizeof(long))
```

### 辅助宏的实现

```c
#if __GNUC__ < 4 || (__GNUC__ == 4 && __GNUC_MINOR__ < 1)
/* Technically wrong, but this avoids compilation errors on some gcc
   versions. */
#define BITOP_ADDR(x) "=m" (*(volatile long *) (x))
#else
#define BITOP_ADDR(x) "+m" (*(volatile long *) (x))
#endif

#define ADDR                            BITOP_ADDR(addr)

/*
 * We do the locked ops that don't return the old value as
 * a mask operation on a byte.
 */
#define IS_IMMEDIATE(nr)                (__builtin_constant_p(nr))
#define CONST_MASK_ADDR(nr, addr)       BITOP_ADDR((void *)(addr) + ((nr)>>3))
#define CONST_MASK(nr)                  (1 << ((nr) & 7))
```

`IS_IMMEDIATE` 宏判断 `nr` 是否是整数常量。如果把位图看成 `8` 列的二维数组，那么 `CONST_MASK_ADDR` 宏负责计算行号，`CONST_MASK` 宏负责计算列号。

### 设置位的实现

`arch/x86/include/asm/bitops.h`：

```c
/**
 * set_bit - Atomically set a bit in memory
 * @nr: the bit to set
 * @addr: the address to start counting from
 *
 * This function is atomic and may not be reordered.  See __set_bit()
 * if you do not require the atomic guarantees.
 *
 * Note: there are no guarantees that this function will not be reordered
 * on non x86 architectures, so if you are writing portable code,
 * make sure not to rely on its reordering guarantees.
 *
 * Note that @nr may be almost arbitrarily large; this function is not
 * restricted to acting on a single-word quantity.
 */
static __always_inline void
set_bit(unsigned int nr, volatile unsigned long *addr)
{
    if (IS_IMMEDIATE(nr)) {
        asm volatile(LOCK_PREFIX "orb %1,%0"
            : CONST_MASK_ADDR(nr, addr)
            : "iq" ((u8)CONST_MASK(nr))
            : "memory");
    } else {
        asm volatile(LOCK_PREFIX "bts %1,%0"
            : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
    }
}
```

`LOCK_PREFIX` 的实现原理涉及锁，见 [TODO](/posts/kernel/todo)。

查阅[相关手册](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html#Machine-Constraints)，可以知道这里 `q` 修饰符代表使用 8 位寄存器（例如 `al`、`bl` 等）。

`bts` 指令（Bit Test and Set）是位测试并置位指令，此处 `bts %1,%0` 即先测试 `%0` 内存处的 `%1` 位的值，存入 `CF` 标志寄存器，再将该位置 `1`。

### 清除位的实现

```c
/**
 * clear_bit - Clears a bit in memory
 * @nr: Bit to clear
 * @addr: Address to start counting from
 *
 * clear_bit() is atomic and may not be reordered.  However, it does
 * not contain a memory barrier, so if it is used for locking purposes,
 * you should call smp_mb__before_clear_bit() and/or smp_mb__after_clear_bit()
 * in order to ensure changes are visible on other processors.
 */
static __always_inline void
clear_bit(int nr, volatile unsigned long *addr)
{
    if (IS_IMMEDIATE(nr)) {
        asm volatile(LOCK_PREFIX "andb %1,%0"
            : CONST_MASK_ADDR(nr, addr)
            : "iq" ((u8)~CONST_MASK(nr)));
    } else {
        asm volatile(LOCK_PREFIX "btr %1,%0"
            : BITOP_ADDR(addr)
            : "Ir" (nr));
    }
}
```

`btr` 指令（Bit Test and Reset）是位测试并复位指令。

### 改变位的实现

```c
/**
 * change_bit - Toggle a bit in memory
 * @nr: Bit to change
 * @addr: Address to start counting from
 *
 * change_bit() is atomic and may not be reordered.
 * Note that @nr may be almost arbitrarily large; this function is not
 * restricted to acting on a single-word quantity.
 */
static inline void change_bit(int nr, volatile unsigned long *addr)
{
    if (IS_IMMEDIATE(nr)) {
        asm volatile(LOCK_PREFIX "xorb %1,%0"
            : CONST_MASK_ADDR(nr, addr)
            : "iq" ((u8)CONST_MASK(nr)));
    } else {
        asm volatile(LOCK_PREFIX "btc %1,%0"
            : BITOP_ADDR(addr)
            : "Ir" (nr));
    }
}
```

`btc` 指令（Bit Test and Complement）是位测试并取反指令。

### 测试并设置位的实现

```c
/**
 * test_and_set_bit - Set a bit and return its old value
 * @nr: Bit to set
 * @addr: Address to count from
 *
 * This operation is atomic and cannot be reordered.
 * It also implies a memory barrier.
 */
static inline int test_and_set_bit(int nr, volatile unsigned long *addr)
{
    int oldbit;

    asm volatile(LOCK_PREFIX "bts %2,%1\n\t"
             "sbb %0,%0" : "=r" (oldbit), ADDR : "Ir" (nr) : "memory");

    return oldbit;
}
```

这里很巧妙地借助了 `sbb` 指令执行带借位减法。如果 `CF` 标志被置位，函数返回 `-1`，反之返回 `0`。

### 测试并清除位的实现

```c
/**
 * test_and_clear_bit - Clear a bit and return its old value
 * @nr: Bit to clear
 * @addr: Address to count from
 *
 * This operation is atomic and cannot be reordered.
 * It also implies a memory barrier.
 */
static inline int test_and_clear_bit(int nr, volatile unsigned long *addr)
{
    int oldbit;

    asm volatile(LOCK_PREFIX "btr %2,%1\n\t"
             "sbb %0,%0"
             : "=r" (oldbit), ADDR : "Ir" (nr) : "memory");

    return oldbit;
}
```

### 测试并改变位的实现

```c
/**
 * test_and_change_bit - Change a bit and return its old value
 * @nr: Bit to change
 * @addr: Address to count from
 *
 * This operation is atomic and cannot be reordered.
 * It also implies a memory barrier.
 */
static inline int test_and_change_bit(int nr, volatile unsigned long *addr)
{
    int oldbit;

    asm volatile(LOCK_PREFIX "btc %2,%1\n\t"
             "sbb %0,%0"
             : "=r" (oldbit), ADDR : "Ir" (nr) : "memory");

    return oldbit;
}
```

### 测试位的实现

```c
static __always_inline int constant_test_bit(unsigned int nr, const volatile unsigned long *addr)
{
    return ((1UL << (nr % BITS_PER_LONG)) &
        (((unsigned long *)addr)[nr / BITS_PER_LONG])) != 0;
}

static inline int variable_test_bit(int nr, volatile const unsigned long *addr)
{
    int oldbit;

    asm volatile("bt %2,%1\n\t"
             "sbb %0,%0"
             : "=r" (oldbit)
             : "m" (*(unsigned long *)addr), "Ir" (nr));

    return oldbit;
}

#if 0 /* Fool kernel-doc since it doesn't do macros yet */
/**
 * test_bit - Determine whether a bit is set
 * @nr: bit number to test
 * @addr: Address to start counting from
 */
static int test_bit(int nr, const volatile unsigned long *addr);
#endif

#define test_bit(nr, addr)               \
    (__builtin_constant_p((nr))          \
     ? constant_test_bit((nr), (addr))   \
     : variable_test_bit((nr), (addr)))
```

`bt` 指令（Bit Test）是位测试指令。
