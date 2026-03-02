---
title: "内核队列"
date: 2020-10-10
description: ""
menu:
  sidebar:
    name: "内核队列"
    identifier: kernel-data-structure-kfifo
    parent: kernel-data-structure
    weight: 100
tags: ["Linux 内核", "内核数据结构", "队列"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/zh-cn/posts/kernel/kernel)，GCC 扩展语法和内核数据结构部分，简单介绍 Linux 内核队列。

<!--more-->

## 内核队列的使用

内核通用队列的实现称为 `kfifo`，头文件为 `include/linux/kfifo.h`。

### 创建队列

创建并初始化一个大小为 `size` 的 `kfifo`，使用 `gfp_mask` 分配内存（成功则返回 `0`，反之返回一个负数错误码）：

```c
int kfifo_alloc(struct kfifo *fifo, unsigned int size, gfp_t gfp_mask);
```

创建并初始化一个 `kfifo` 对象，使用 `buffer` 指向的 `size` 大小的内存：

```c
void kfifo_init(struct kfifo *fifo, void *buffer, unsigned int size);
```

静态声明 `kfifo`（不常用）：

```c
DECLARE_KFIFO(name, size);
INIT_KFIFO(name);
```

**以上所有的 `size` 必须为 `2` 的幂。**

### 插入队列数据

```c
unsigned int kfifo_in(struct kfifo *fifo, const void *from, unsigned int len);
```

该函数把 `from` 指针所指的 `len` 字节数据拷贝到 `fifo` 所指的队列中。如果成功，返回推入数据的字节大小。如果队列中的空闲字节小于 `len`，返回值可能小于 `len`，甚至会返回 `0`，这时意味着没有任何数据被推入。

### 摘取队列数据

```c
unsigned int kfifo_out(struct kfifo *fifo, void *to, unsigned int len);
```

该函数从 `fifo` 所指向的队列中拷贝出长度为 `len` 字节的数据到 `to` 所指的缓冲中。如果成功，返回拷贝的数据长度。如果队列中数据大小小于 `len`，则该函数拷贝出的数据必然小于需要的数据大小。

当数据被摘取后，数据就不再存在于队列之中。这是队列操作的常用方式。不过如果仅仅想“偷窥”队列中的数据，而不真想删除它，可以使用下面的方法:

```c
unsigned int kfifo_out_peek(struct kfifo *fifo, void *to, unsigned int len, unsigned offset);
```

该函数和 `kffo_out()` 类似，但出口偏移不增加，而且摘取的数据仍然可被下次 `kfifo_out()` 获得。参数 `offset` 指向队列中的索引位置，如果该参数为 `0`，则读队列头。

### 获取队列长度

获得用于存储 `kfifo` 队列的空间的总体大小：

```c
unsigned int kfifo_size(struct kfifo *fifo);
```

获得 `kfifo` 队列中已推入的数据大小：

```c
unsigned int kfifo_len(struct kfifo *fifo);
```

获得 `kfifo` 队列中还有多少可用空间：

```c
unsigned int kfifo_avail(struct kfifo *fifo);
```

**以上所有函数的返回值以字节为单位。**

判断给定的 `kfifo` 是否为空的，若是则返回非 `0` 值，反之返回 `0`：

```c
int kfifo_is_empty(struct kfifo *fifo);
```

判断给定的 `kfifo` 是否为空的，若是则返回非 `0` 值，反之返回 `0`：

```c
int kfifo_is_full(struct kfifo *fifo);
```

### 重置队列

```c
void kfifo_reset(struct kfifo *fifo);
```

### 撤销队列

若通过 `kfifo_alloc()` 创建的队列，用下面的函数撤销：

```c
void kfifo_free(struct kfifo *fifo);
```

若通过 `kfifo_init()` 创建的队列，需要手动释放相应的内存。

### 队列使用实例

```c
struct kfifo fifo;
int ret;
unsigned int i;
unsigned int val;

// 创建一个大小为 PAGE_SIZE 的队列 fifo
ret = kfifo_alloc(&kifo, PAGE_SIZE, GFP_KERNEL);
if (ret)
    return ret;

// 将 [0,32) 压入到名为 fifo 的 kfifo 中
for(i = 0; i < 32; i++)
    kfifo_in(fifo, &i, sizeof(i));

// 查看队列的第一个元素
ret = kfifo_out_peek(fifo, &val, sizeof(val), 0);
if (ret != sizeof(val))
    return -EINVAL;
printk(KERN_INFO "%u\n", val); // 应该输出 0

// 摘取并打印 kfifo 中的所有元素
while (kfifo_avail(fifo)) {
    ret = kfifo_out(fifo, &val, sizeof(val));
    if (ret != sizeof(val))
        return -EINVAL;
    printk(KERN_INFO "%u\n", val); // 应该按序输出 0 到 31
}
```

## 内核队列的实现

内核队列的实现均位于 `kernel/kfifo.c` 和 `include/linux/kfifo.h`，下面列出常用函数/宏的实现。

### 队列结构体的实现

```c
struct kfifo {
    unsigned char *buffer; /* the buffer holding the data */
    unsigned int size;     /* the size of the allocated buffer */
    unsigned int in;       /* data is added at offset (in % size) */
    unsigned int out;      /* data is extracted from off. (out % size) */
};
```

显然，内核队列是一个循环队列。下面我们把 `in` 称为队尾指针，`out` 称为队头指针，`buffer` 视为缓冲区数组。

传统的队列存储的数据类型往往是已知的，入队、出队一个元素对应缓冲区数组下标加减 `1`。而为了通用性，内核队列把缓冲区数组声明为 `char` 类型，对应的入队、出队需要通过元素的大小来计算下标的值（如 `int` 类型的一个元素入队、出队对应缓冲区数组下标加减 `4`）。

既然是循环队列，就存在区分队满和队空的情况，传统的解决方法就几种：一是记录队列的元素个数或记录队列的总大小；二是牺牲一个存储单元来用作判断；三是通过一个额外的标志来判断队满或队空。此处采用了第一种方法，用 `size` 记录了缓冲区数组的大小。

### 帮助函数/宏的实现

```c
/* helper macro */
#define __kfifo_initializer(s, b) \
    (struct kfifo) { \
        .size   = s, \
        .in     = 0, \
        .out    = 0, \
        .buffer = b \
    }

/*
 * __kfifo_add_out internal helper function for updating the out offset
 */
static inline void __kfifo_add_out(struct kfifo *fifo,
                unsigned int off)
{
    smp_mb();
    fifo->out += off;
}

/*
 * __kfifo_add_in internal helper function for updating the in offset
 */
static inline void __kfifo_add_in(struct kfifo *fifo,
                unsigned int off)
{
    smp_wmb();
    fifo->in += off;
}

/*
 * __kfifo_off internal helper function for calculating the index of a
 * given offeset
 */
static inline unsigned int __kfifo_off(struct kfifo *fifo, unsigned int off)
{
    return off & (fifo->size - 1);
}

static inline void __kfifo_in_data(struct kfifo *fifo,
        const void *from, unsigned int len, unsigned int off)
{
    unsigned int l;

    /*
     * Ensure that we sample the fifo->out index -before- we
     * start putting bytes into the kfifo.
     */

    smp_mb();

    off = __kfifo_off(fifo, fifo->in + off);

    /* first put the data starting from fifo->in to buffer end */
    l = min(len, fifo->size - off);
    memcpy(fifo->buffer + off, from, l);

    /* then put the rest (if any) at the beginning of the buffer */
    memcpy(fifo->buffer, from + l, len - l);
}

static inline void __kfifo_out_data(struct kfifo *fifo,
        void *to, unsigned int len, unsigned int off)
{
    unsigned int l;

    /*
     * Ensure that we sample the fifo->in index -before- we
     * start removing bytes from the kfifo.
     */

    smp_rmb();

    off = __kfifo_off(fifo, fifo->out + off);

    /* first get the data from fifo->out until the end of the buffer */
    l = min(len, fifo->size - off);
    memcpy(to, fifo->buffer + off, l);

    /* then get the rest (if any) from the beginning of the buffer */
    memcpy(to + l, fifo->buffer, len - l);
}
```

- `__kfifo_initializer()`：静态初始化队列。
- `__kfifo_add_out()`：增加队头指针。
- `__kfifo_add_in()`：增加队尾指针。
- `__kfifo_off()`：计算 `in % size` 或 `out % size`。
- `__kfifo_in_data()`：从 `from` 拷贝 `len` 大小的数据到队列头部偏移 `off` 的位置。
- `__kfifo_out_data()`：从队列头部偏移 `off` 的位置拷贝 `len` 大小的数据到 `to`。

这里先忽略 `smp_` 开头的函数，它们是内存屏障，我把内核队列作为介绍内存屏障的例子，见 [TODO](/zh-cn/posts/kernel/todo)。

对于 `__kfifo_off()` 中的计算，其实是用了这么一个结论：如果 `m` 是 `2` 的幂，那么 `n % m` 等价于 `n & (m - 1)`。这个结论其实不难理解，我们来看一个简单的例子。假设所有的数占用 `8` 位，取 `m` 为 `4`，那么 `m` 的二进制为 `0000 0100`，`m - 1` 的二进制为 `0000 0011` 。取 `n` 为 `3`，即 `n` 的二进制为 `0000 0011`，那么 `n & (m - 1)` 的结果为 `0000 0011`。取 `n` 为 `5`，即 `n` 的二进制为 `0000 0101`，那么 `n & (m - 1)` 的结果为 `0000 0001`。不难发现 `n & (m - 1)` 的意义是把 `n` 的高位清零且低位不变，这和 `n % m` 是等价的（具体的数学证明显然不属于本学习笔记的范畴）。通过使用与运算来代替模运算，可以有效缩短运算时间。

`__kfifo_in_data()` 和 `__kfifo_out_data()` 相对复杂一些。因为是循环队列，所以更要注意要缓冲区数组边界的判断。对于 `__kfifo_in_data()`，先是拷贝数据到队尾指针偏移处，若拷贝至数组尾部仍然不够，再拷贝剩下的内容到数组头部。对于 `__kfifo_out_data()`，先是从队头指针偏移处拷贝数据，若拷贝至数组尾部仍然不够，再从数组头部拷贝剩下的内容。此处的实现考虑了的偏移 `off` 和数据大小 `len`，而对应常规的入队、出队单个元素的操作，偏移 `off` 总为 `0`，且 `len` 总为单个元素的大小，这点可以从下面的[插入队列数据的实现](#插入队列数据的实现)和[摘取队列数据的实现](#摘取队列数据的实现)中看到。

### 获取队列长度的实现

```c
/**
 * kfifo_size - returns the size of the fifo in bytes
 * @fifo: the fifo to be used.
 */
static inline __must_check unsigned int kfifo_size(struct kfifo *fifo)
{
    return fifo->size;
}

/**
 * kfifo_len - returns the number of used bytes in the FIFO
 * @fifo: the fifo to be used.
 */
static inline unsigned int kfifo_len(struct kfifo *fifo)
{
    register unsigned int out;

    out = fifo->out;
    smp_rmb();
    return fifo->in - out;
}

/**
 * kfifo_is_empty - returns true if the fifo is empty
 * @fifo: the fifo to be used.
 */
static inline __must_check int kfifo_is_empty(struct kfifo *fifo)
{
    return fifo->in == fifo->out;
}

/**
 * kfifo_is_full - returns true if the fifo is full
 * @fifo: the fifo to be used.
 */
static inline __must_check int kfifo_is_full(struct kfifo *fifo)
{
    return kfifo_len(fifo) == kfifo_size(fifo);
}

/**
 * kfifo_avail - returns the number of bytes available in the FIFO
 * @fifo: the fifo to be used.
 */
static inline __must_check unsigned int kfifo_avail(struct kfifo *fifo)
{
    return kfifo_size(fifo) - kfifo_len(fifo);
}
```

### 重置队列的实现

```c
/**
 * kfifo_reset - removes the entire FIFO contents
 * @fifo: the fifo to be emptied.
 */
static inline void kfifo_reset(struct kfifo *fifo)
{
    fifo->in = fifo->out = 0;
}
```

### 创建队列的实现

```c
/**
 * INIT_KFIFO - Initialize a kfifo declared by DECLARE_KFIFO
 * @name: name of the declared kfifo datatype
 */
#define INIT_KFIFO(name) \
    name = __kfifo_initializer(sizeof(name##kfifo_buffer) - \
                sizeof(struct kfifo), \
                name##kfifo_buffer + sizeof(struct kfifo))

/**
 * DEFINE_KFIFO - macro to define and initialize a kfifo
 * @name: name of the declared kfifo datatype
 * @size: size of the fifo buffer. Must be a power of two.
 *
 * Note1: the macro can be used for global and local kfifo data type variables
 * Note2: the macro creates two objects:
 *  A kfifo object with the given name and a buffer for the kfifo
 *  object named name##kfifo_buffer
 */
#define DEFINE_KFIFO(name, size) \
    unsigned char name##kfifo_buffer[size]; \
    struct kfifo name = __kfifo_initializer(size, name##kfifo_buffer)

static void _kfifo_init(struct kfifo *fifo, void *buffer,
        unsigned int size)
{
    fifo->buffer = buffer;
    fifo->size = size;

    kfifo_reset(fifo);
}

/**
 * kfifo_init - initialize a FIFO using a preallocated buffer
 * @fifo: the fifo to assign the buffer
 * @buffer: the preallocated buffer to be used.
 * @size: the size of the internal buffer, this has to be a power of 2.
 *
 */
void kfifo_init(struct kfifo *fifo, void *buffer, unsigned int size)
{
    /* size must be a power of 2 */
    BUG_ON(!is_power_of_2(size));

    _kfifo_init(fifo, buffer, size);
}
EXPORT_SYMBOL(kfifo_init);

/**
 * kfifo_alloc - allocates a new FIFO internal buffer
 * @fifo: the fifo to assign then new buffer
 * @size: the size of the buffer to be allocated, this have to be a power of 2.
 * @gfp_mask: get_free_pages mask, passed to kmalloc()
 *
 * This function dynamically allocates a new fifo internal buffer
 *
 * The size will be rounded-up to a power of 2.
 * The buffer will be release with kfifo_free().
 * Return 0 if no error, otherwise the an error code
 */
int kfifo_alloc(struct kfifo *fifo, unsigned int size, gfp_t gfp_mask)
{
    unsigned char *buffer;

    /*
     * round up to the next power of 2, since our 'let the indices
     * wrap' technique works only in this case.
     */
    if (!is_power_of_2(size)) {
        BUG_ON(size > 0x80000000);
        size = roundup_pow_of_two(size);
    }

    buffer = kmalloc(size, gfp_mask);
    if (!buffer) {
        _kfifo_init(fifo, NULL, 0);
        return -ENOMEM;
    }

    _kfifo_init(fifo, buffer, size);

    return 0;
}
EXPORT_SYMBOL(kfifo_alloc);
```

### 插入队列数据的实现

```c
/**
 * kfifo_in - puts some data into the FIFO
 * @fifo: the fifo to be used.
 * @from: the data to be added.
 * @len: the length of the data to be added.
 *
 * This function copies at most @len bytes from the @from buffer into
 * the FIFO depending on the free space, and returns the number of
 * bytes copied.
 *
 * Note that with only one concurrent reader and one concurrent
 * writer, you don't need extra locking to use these functions.
 */
unsigned int kfifo_in(struct kfifo *fifo, const void *from,
                unsigned int len)
{
    len = min(kfifo_avail(fifo), len);

    __kfifo_in_data(fifo, from, len, 0);
    __kfifo_add_in(fifo, len);
    return len;
}
EXPORT_SYMBOL(kfifo_in);
```

### 摘取队列数据的实现

```c
/**
 * kfifo_out - gets some data from the FIFO
 * @fifo: the fifo to be used.
 * @to: where the data must be copied.
 * @len: the size of the destination buffer.
 *
 * This function copies at most @len bytes from the FIFO into the
 * @to buffer and returns the number of copied bytes.
 *
 * Note that with only one concurrent reader and one concurrent
 * writer, you don't need extra locking to use these functions.
 */
unsigned int kfifo_out(struct kfifo *fifo, void *to, unsigned int len)
{
    len = min(kfifo_len(fifo), len);

    __kfifo_out_data(fifo, to, len, 0);
    __kfifo_add_out(fifo, len);

    return len;
}
EXPORT_SYMBOL(kfifo_out);

/**
 * kfifo_out_peek - copy some data from the FIFO, but do not remove it
 * @fifo: the fifo to be used.
 * @to: where the data must be copied.
 * @len: the size of the destination buffer.
 * @offset: offset into the fifo
 *
 * This function copies at most @len bytes at @offset from the FIFO
 * into the @to buffer and returns the number of copied bytes.
 * The data is not removed from the FIFO.
 */
unsigned int kfifo_out_peek(struct kfifo *fifo, void *to, unsigned int len,
                unsigned offset)
{
    len = min(kfifo_len(fifo), len + offset);

    __kfifo_out_data(fifo, to, len, offset);
    return len;
}
EXPORT_SYMBOL(kfifo_out_peek);
```

### 撤销队列的实现

```c
/**
 * kfifo_free - frees the FIFO internal buffer
 * @fifo: the fifo to be freed.
 */
void kfifo_free(struct kfifo *fifo)
{
    kfree(fifo->buffer);
    _kfifo_init(fifo, NULL, 0);
}
EXPORT_SYMBOL(kfifo_free);
```
