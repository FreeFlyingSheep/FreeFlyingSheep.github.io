---
title: "非连续页框的管理"
date: 2021-04-26
description: ""
menu:
  sidebar:
    name: "非连续页框的管理"
    identifier: kernel-memory-uncontinuous
    parent: kernel-memory
    weight: 100
tags: ["Linux 内核", "内存管理", "非连续页框的管理"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，内存管理部分，简单介绍非连续页框的管理。

<!--more-->

## 非连续内存区的线性地址空间

非连续内存区的线性地址空间的线性地址空间从 `VMALLOC_START` 开始，到 `VMALLOC_END` 结束，这个宏显然是体系结构相关的，下面以 32 位的 x86 为例。

`arch/x86/include/asm/pgtable_32_types.h`：

```c
#define VMALLOC_OFFSET  (8 * 1024 * 1024)

#define VMALLOC_START   ((unsigned long)high_memory + VMALLOC_OFFSET)

# define VMALLOC_END    (PKMAP_BASE - 2 * PAGE_SIZE)
```

`high_memory` 对应直接映射的物理内存的末尾，`VMALLOC_OFFSET` 是一个 `8MB` 的安全空间，用于与直接映射的物理内存区域隔开。`PKMAP_BASE` 开始是永久内核映射的线性地址，`VMALLOC_END` 与之隔了个 `8KB` 的安全空间（默认页大小是 `4KB`）。

高端内存的布局大致如下（`arch/x86/include/asm/highmem.h` 中的注释）：

```c
/*
 * Right now we initialize only a single pte table. It can be extended
 * easily, subsequent pte tables have to be allocated in one physical
 * chunk of RAM.
 */
/*
 * Ordering is:
 *
 * FIXADDR_TOP
 *                      fixed_addresses
 * FIXADDR_START
 *                      temp fixed addresses
 * FIXADDR_BOOT_START
 *                      Persistent kmap area
 * PKMAP_BASE
 * VMALLOC_END
 *                      Vmalloc area
 * VMALLOC_START
 * high_memory
 */
```

## 非连续内存区的描述符

每个非连续内存区对应一个 `vm_struct` 描述符。

`include/linux/vmalloc.h`：

```c
struct vm_struct {
    struct vm_struct    *next;
    void                *addr;
    unsigned long       size;
    unsigned long       flags;
    struct page         **pages;
    unsigned int        nr_pages;
    unsigned long       phys_addr;
    void                *caller;
};
```

- `next`：指向下一个 `vm_struct` 的指针。
- `addr`：内存区内第一个内存单元的线性地址。
- `size`：内存区的大小加上 `4096` （安全区间为页大小）。
- `flags`：映射的内存类型。
- `pages`：指向页框描述符指针数组的指针（页框描述符是 `struct page`）。
- `nr_pages`：页框描述符指针数组中填充的页的个数。
- `phys_addr`：映射硬件设备的 I/O 共享内存（没映射则为 `0`）
- `caller`：调用者的返回地址（`__vmalloc_node()` 的注释是这么写的，具体用途没研究，这几本书都没介绍，所以我也准备跳过了）

其中，映射的内存类型定义如下。

`include/linux/vmalloc.h`：

```c
/* bits in flags of vmalloc's vm_struct below */
#define VM_IOREMAP  0x00000001  /* ioremap() and friends */
#define VM_ALLOC    0x00000002  /* vmalloc() */
#define VM_MAP      0x00000004  /* vmap()ed pages */
#define VM_USERMAP  0x00000008  /* suitable for remap_vmalloc_range */
#define VM_VPAGES   0x00000010  /* buffer for pages was vmalloc'ed */
/* bits [20..32] reserved for arch specific ioremap internals */
```

## 分配非连续内存区

分配非连续内存区使用 `vmalloc()` 函数。

`mm/vmalloc.c`：

```c
/**
 *  __vmalloc_node  -  allocate virtually contiguous memory
 *  @size:      allocation size
 *  @align:     desired alignment
 *  @gfp_mask:  flags for the page level allocator
 *  @prot:      protection mask for the allocated pages
 *  @node:      node to use for allocation or -1
 *  @caller:    caller's return address
 *
 *  Allocate enough pages to cover @size from the page level
 *  allocator with @gfp_mask flags.  Map them into contiguous
 *  kernel virtual space, using a pagetable protection of @prot.
 */
static void *__vmalloc_node(unsigned long size, unsigned long align,
                            gfp_t gfp_mask, pgprot_t prot,
                            int node, void *caller)
{
    struct vm_struct *area;
    void *addr;
    unsigned long real_size = size;

    size = PAGE_ALIGN(size);
    if (!size || (size >> PAGE_SHIFT) > totalram_pages)
        return NULL;

    area = __get_vm_area_node(size, align, VM_ALLOC, VMALLOC_START,
                  VMALLOC_END, node, gfp_mask, caller);

    if (!area)
        return NULL;

    addr = __vmalloc_area_node(area, gfp_mask, prot, node, caller);

    /*
     * A ref_count = 3 is needed because the vm_struct and vmap_area
     * structures allocated in the __get_vm_area_node() function contain
     * references to the virtual address of the vmalloc'ed block.
     */
    kmemleak_alloc(addr, real_size, 3, gfp_mask);

    return addr;
}

/**
 *  vmalloc  -  allocate virtually contiguous memory
 *  @size:      allocation size
 *  Allocate enough pages to cover @size from the page level
 *  allocator and map them into contiguous kernel virtual space.
 *
 *  For tight control over page level allocator and protection flags
 *  use __vmalloc() instead.
 */
void *vmalloc(unsigned long size)
{
    return __vmalloc_node(size, 1, GFP_KERNEL | __GFP_HIGHMEM, PAGE_KERNEL,
                    -1, __builtin_return_address(0));
}
EXPORT_SYMBOL(vmalloc);
```

`__vmalloc_node()` 主要进行如下操作：

1. 通过 `PAGE_ALIGN()` 宏将 `size` 对齐到页大小。
2. 通过 `__get_vm_area_node()` 创建新的描述符。
3. 通过 `__vmalloc_area_node()` 分配内存空间。
4. 通过 `kmemleak_alloc()` 通报内存分配。

### `PAGE_ALIGN()`

`PAGE_ALIGN()` 的原理比较简单，挨个宏找下去就行了。

`include/linux/mm.h`：

```c
/* to align the pointer to the (next) page boundary */
#define PAGE_ALIGN(addr) ALIGN(addr, PAGE_SIZE)
```

`include/linux/kernel.h`：

```c
#define ALIGN(x,a)              __ALIGN_MASK(x,(typeof(x))(a)-1)
#define __ALIGN_MASK(x,mask)    (((x)+(mask))&~(mask))
```

### `__get_vm_area_node()`

`mm/vmalloc.c`：

```c
static struct vm_struct *__get_vm_area_node(unsigned long size,
        unsigned long align, unsigned long flags, unsigned long start,
        unsigned long end, int node, gfp_t gfp_mask, void *caller)
{
    static struct vmap_area *va;
    struct vm_struct *area;

    BUG_ON(in_interrupt());
    if (flags & VM_IOREMAP) {
        int bit = fls(size);

        if (bit > IOREMAP_MAX_ORDER)
            bit = IOREMAP_MAX_ORDER;
        else if (bit < PAGE_SHIFT)
            bit = PAGE_SHIFT;

        align = 1ul << bit;
    }

    size = PAGE_ALIGN(size);
    if (unlikely(!size))
        return NULL;

    area = kzalloc_node(sizeof(*area), gfp_mask & GFP_RECLAIM_MASK, node);
    if (unlikely(!area))
        return NULL;

    /*
     * We always allocate a guard page.
     */
    size += PAGE_SIZE;

    va = alloc_vmap_area(size, align, start, end, node, gfp_mask);
    if (IS_ERR(va)) {
        kfree(area);
        return NULL;
    }

    insert_vmalloc_vm(area, va, flags, caller);
    return area;
}
```

`__get_vm_area_node()` 主要用于创建新的描述符，并把该描述符插入到相关的内核数据结构中，具体细节比较复杂，我不准备看了。

### `__vmalloc_area_node()`

`mm/vmalloc.c`：

```c
static void *__vmalloc_area_node(struct vm_struct *area, gfp_t gfp_mask,
                                 pgprot_t prot, int node, void *caller)
{
    struct page **pages;
    unsigned int nr_pages, array_size, i;
    gfp_t nested_gfp = (gfp_mask & GFP_RECLAIM_MASK) | __GFP_ZERO;

    nr_pages = (area->size - PAGE_SIZE) >> PAGE_SHIFT;
    array_size = (nr_pages * sizeof(struct page *));

    area->nr_pages = nr_pages;
    /* Please note that the recursion is strictly bounded. */
    if (array_size > PAGE_SIZE) {
        pages = __vmalloc_node(array_size, 1, nested_gfp|__GFP_HIGHMEM,
                PAGE_KERNEL, node, caller);
        area->flags |= VM_VPAGES;
    } else {
        pages = kmalloc_node(array_size, nested_gfp, node);
    }
    area->pages = pages;
    area->caller = caller;
    if (!area->pages) {
        remove_vm_area(area->addr);
        kfree(area);
        return NULL;
    }

    for (i = 0; i < area->nr_pages; i++) {
        struct page *page;

        if (node < 0)
            page = alloc_page(gfp_mask);
        else
            page = alloc_pages_node(node, gfp_mask, 0);

        if (unlikely(!page)) {
            /* Successfully allocated i pages, free them in __vunmap() */
            area->nr_pages = i;
            goto fail;
        }
        area->pages[i] = page;
    }

    if (map_vm_area(area, prot, &pages))
        goto fail;
    return area->addr;

fail:
    vfree(area->addr);
    return NULL;
}
```

`__vmalloc_area_node()` 和 `__vmalloc_node()` 互相递归，当分配的页描述符指针数组小于一页时，`__vmalloc_area_node()` 调用 `kmalloc_node()` 来分配（`kmalloc_node()` 会通过 `kmalloc()` 分配内存），结束递归。

页描述符指针数组的元素指向 `alloc_pages_node()` 分配的页（`alloc_page()` 最终也会调用 `alloc_pages_node()`，见[请求页框](/posts/kernel/memory/continuous#请求页框)），完成填充。

完成分配后，通过 `map_vm_area()` 来修改页表项，这个函数的细节比较复杂，暂时不准备去看了。

### kmemleak_alloc()

`kmemleak_alloc()` 是 kmemleak 的 API，kmemleak 主要用于检测内存泄漏，相关内容不属于本系列学习笔记的范畴。

## 释放非连续内存区

释放非连续内存区使用 `vfree()` 函数。

`mm/vmalloc.c`：

```c
static void __vunmap(const void *addr, int deallocate_pages)
{
    struct vm_struct *area;

    if (!addr)
        return;

    if ((PAGE_SIZE-1) & (unsigned long)addr) {
        WARN(1, KERN_ERR "Trying to vfree() bad address (%p)\n", addr);
        return;
    }

    area = remove_vm_area(addr);
    if (unlikely(!area)) {
        WARN(1, KERN_ERR "Trying to vfree() nonexistent vm area (%p)\n",
                addr);
        return;
    }

    debug_check_no_locks_freed(addr, area->size);
    debug_check_no_obj_freed(addr, area->size);

    if (deallocate_pages) {
        int i;

        for (i = 0; i < area->nr_pages; i++) {
            struct page *page = area->pages[i];

            BUG_ON(!page);
            __free_page(page);
        }

        if (area->flags & VM_VPAGES)
            vfree(area->pages);
        else
            kfree(area->pages);
    }

    kfree(area);
    return;
}

/**
 *  vfree  -    release memory allocated by vmalloc()
 *  @addr:      memory base address
 *
 *  Free the virtually continuous memory area starting at @addr, as
 *  obtained from vmalloc(), vmalloc_32() or __vmalloc(). If @addr is
 *  NULL, no operation is performed.
 *
 *  Must not be called in interrupt context.
 */
void vfree(const void *addr)
{
    BUG_ON(in_interrupt());

    kmemleak_free(addr);

    __vunmap(addr, 1);
}
EXPORT_SYMBOL(vfree);
```

`__vunmap()` 主要进行如下操作（进行和分配顺序相反的操作）：

1. 通过 `remove_vm_area()` 清楚相应的页表项。
2. 通过 `__free_page()` 释放分配的页。
3. 通过 `kfree()` 释放描述符。

但这里面的细节比较复杂，我暂时不准备看了。
