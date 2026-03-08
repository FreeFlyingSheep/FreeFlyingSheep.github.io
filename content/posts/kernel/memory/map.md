---
title: "内存映射"
date: 2021-04-26
description: ""
menu:
  sidebar:
    name: "内存映射"
    identifier: kernel-memory-map
    parent: kernel-memory
    weight: 100
tags: ["Linux 内核", "内存管理", "内存映射"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，内存管理部分，简单介绍内存映射。

<!--more-->

## 高端内存页框的内存映射简介

内核可以采用三种不同的机制讲页框映射到高端内存：永久内核映射、临时内核映射和非连续内存分配。其中，非连续内存分配的部分见[非连续内存分配](/posts/kernel/memory/uncontinuous)。

## 永久内核映射

永久内核映射允许内核建立高端页框到内核地址空间的长期映射。当空闲页表项不存在（高端内存上没有页表项可以用作页框的窗口）时，可能会阻塞当前进程。因此，它不能用于中断处理程序和可延迟函数。

### 页表

永久映射使用主内核页表中的一个专门页表。

`mm/highmem.c`：

```c
pte_t * pkmap_page_table;
```

页表的表项数由 `LAST_PKMAP` 宏决定，该宏是体系结构相关的。

`arch/x86/include/asm/pgtable_32_types.h`：

```c
#ifdef CONFIG_X86_PAE
#define LAST_PKMAP 512
#else
#define LAST_PKMAP 1024
#endif
```

页表映射的线性地址从 `PKMAP_BASE`（该宏的实现与 vsyscall 有关，不属于本系列学习笔记的范畴）开始，每个表项都有一个计数器，存放于数组 `pkmap_count` 中。

`mm/highmem.c`：

```c
static int pkmap_count[LAST_PKMAP];
```

为了页框与映射的线性地址之间的联系，内核使用了 `page_address_htable` 散列表。

`mm/highmem.c`：

```c
static struct page_address_slot {
    struct list_head lh;    /* List of page_address_maps */
    spinlock_t lock;        /* Protect this bucket's list */
} ____cacheline_aligned_in_smp page_address_htable[1<<PA_HASH_ORDER];

static struct page_address_slot *page_slot(struct page *page)
{
    return &page_address_htable[hash_ptr(page, PA_HASH_ORDER)];
}
```

### `kmap()` 函数

`kmap()` 函数用于建立永久内核映射，该函数是体系结构相关的。

`arch/x86/mm/highmem_32.c`：

```c
void *kmap(struct page *page)
{
    might_sleep();
    if (!PageHighMem(page))
        return page_address(page);
    return kmap_high(page);
}
```

`PageHighMem()` 宏定义于 `include/linux/page-flags.h`，该宏会根据内存管理区（Zone）来判断页框是否在高端内存。

如果不在高端内存，则线性地址一定存在，调用 `page_address()` 函数根据物理地址计算出对应的线性地址。

`include/linux/mm.h`：

```c
static __always_inline void *lowmem_page_address(struct page *page)
{
    return __va(page_to_pfn(page) << PAGE_SHIFT);
}
```

`mm/highmem.c`：

```c
/**
 * page_address - get the mapped virtual address of a page
 * @page: &struct page to get the virtual address of
 *
 * Returns the page's virtual address.
 */
void *page_address(struct page *page)
{
    unsigned long flags;
    void *ret;
    struct page_address_slot *pas;

    if (!PageHighMem(page))
        return lowmem_page_address(page);

    pas = page_slot(page);
    ret = NULL;
    spin_lock_irqsave(&pas->lock, flags);
    if (!list_empty(&pas->lh)) {
        struct page_address_map *pam;

        list_for_each_entry(pam, &pas->lh, list) {
            if (pam->page == page) {
                ret = pam->virtual;
                goto done;
            }
        }
    }
done:
    spin_unlock_irqrestore(&pas->lock, flags);
    return ret;
}

EXPORT_SYMBOL(page_address);
```

如果页框在高端内存，则调用 `kmap_high()` 函数。

`mm/highmem.c`：

```c
/**
 * kmap_high - map a highmem page into memory
 * @page: &struct page to map
 *
 * Returns the page's virtual memory address.
 *
 * We cannot call this from interrupts, as it may block.
 */
void *kmap_high(struct page *page)
{
    unsigned long vaddr;

    /*
     * For highmem pages, we can't trust "virtual" until
     * after we have the lock.
     */
    lock_kmap();
    vaddr = (unsigned long)page_address(page);
    if (!vaddr)
        vaddr = map_new_virtual(page);
    pkmap_count[PKMAP_NR(vaddr)]++;
    BUG_ON(pkmap_count[PKMAP_NR(vaddr)] < 2);
    unlock_kmap();
    return (void*) vaddr;
}

EXPORT_SYMBOL(kmap_high);
```

`kmap_high()` 函数首先获取 `lock_kmap` 自旋锁（因为中断处理程序和可延迟函数不能调用 `kmap()`，所以不需要关中断）。

然后通过 `page_address()` 函数来获取页框的线性地址。如果地址不存在，说明未被映射，此时通过 `map_new_virtual()` 函数把页框的物理地址插入到页表项，并在散列表中增加相应的元素。

其次，使页框对应的计数器加一。

最后，释放自旋锁并返回页框映射的线性地址。

`map_new_virtual()` 的实现比较复杂。

`mm/highmem.c`：

```c
static inline unsigned long map_new_virtual(struct page *page)
{
    unsigned long vaddr;
    int count;

start:
    count = LAST_PKMAP;
    /* Find an empty entry */
    for (;;) {
        last_pkmap_nr = (last_pkmap_nr + 1) & LAST_PKMAP_MASK;
        if (!last_pkmap_nr) {
            flush_all_zero_pkmaps();
            count = LAST_PKMAP;
        }
        if (!pkmap_count[last_pkmap_nr])
            break;  /* Found a usable entry */
        if (--count)
            continue;

        /*
         * Sleep for somebody else to unmap their entries
         */
        {
            DECLARE_WAITQUEUE(wait, current);

            __set_current_state(TASK_UNINTERRUPTIBLE);
            add_wait_queue(&pkmap_map_wait, &wait);
            unlock_kmap();
            schedule();
            remove_wait_queue(&pkmap_map_wait, &wait);
            lock_kmap();

            /* Somebody else might have mapped it while we slept */
            if (page_address(page))
                return (unsigned long)page_address(page);

            /* Re-start */
            goto start;
        }
    }
    vaddr = PKMAP_ADDR(last_pkmap_nr);
    set_pte_at(&init_mm, vaddr,
           &(pkmap_page_table[last_pkmap_nr]), mk_pte(page, kmap_prot));

    pkmap_count[last_pkmap_nr] = 1;
    set_page_address(page, (void *)vaddr);

    return vaddr;
}
```

不得不说这个循环的写法有些独特，但大体意思就是遍历 `LAST_PKMAP` 个表项，如果没有找到能用的，阻塞当前进程，等待其他进程释放了一个表项（见[`kunmap()` 函数](#kunmap-函数)）。当前进错被唤醒后，先检查其他进程是否已经映射了该页，是的话直接返回，反之重新开始循环。

下面来看一下具体的遍历过程，通过用 `last_pkmap_nr` 记录上次停止的位置，函数总是从上次停止的地方继续遍历。

找到能用的表项时，跳出循环，将其计数器置 `1`，并把该项插入散列表。

`mm/highmem.c`：

```c
/**
 * set_page_address - set a page's virtual address
 * @page: &struct page to set
 * @virtual: virtual address to use
 */
void set_page_address(struct page *page, void *virtual)
{
    unsigned long flags;
    struct page_address_slot *pas;
    struct page_address_map *pam;

    BUG_ON(!PageHighMem(page));

    pas = page_slot(page);
    if (virtual) {  /* Add */
        BUG_ON(list_empty(&page_address_pool));

        spin_lock_irqsave(&pool_lock, flags);
        pam = list_entry(page_address_pool.next,
                struct page_address_map, list);
        list_del(&pam->list);
        spin_unlock_irqrestore(&pool_lock, flags);

        pam->page = page;
        pam->virtual = virtual;

        spin_lock_irqsave(&pas->lock, flags);
        list_add_tail(&pam->list, &pas->lh);
        spin_unlock_irqrestore(&pas->lock, flags);
    } else {        /* Remove */
        spin_lock_irqsave(&pas->lock, flags);
        list_for_each_entry(pam, &pas->lh, list) {
            if (pam->page == page) {
                list_del(&pam->list);
                spin_unlock_irqrestore(&pas->lock, flags);
                spin_lock_irqsave(&pool_lock, flags);
                list_add_tail(&pam->list, &page_address_pool);
                spin_unlock_irqrestore(&pool_lock, flags);
                goto done;
            }
        }
        spin_unlock_irqrestore(&pas->lock, flags);
    }
done:
    return;
}
```

当 `last_pkmap_nr` 回到最开始的 `0` 时，调用 `flush_all_zero_pkmaps()` 函数来重置计数器，删除散列表中对应的元素，并且刷新 TLB。

`mm/highmem.c`：

```c
static void flush_all_zero_pkmaps(void)
{
    int i;
    int need_flush = 0;

    flush_cache_kmaps();

    for (i = 0; i < LAST_PKMAP; i++) {
        struct page *page;

        /*
         * zero means we don't have anything to do,
         * >1 means that it is still in use. Only
         * a count of 1 means that it is free but
         * needs to be unmapped
         */
        if (pkmap_count[i] != 1)
            continue;
        pkmap_count[i] = 0;

        /* sanity check */
        BUG_ON(pte_none(pkmap_page_table[i]));

        /*
         * Don't need an atomic fetch-and-clear op here;
         * no-one has the page mapped, and cannot get at
         * its virtual address (and hence PTE) without first
         * getting the kmap_lock (which is held here).
         * So no dangers, even with speculative execution.
         */
        page = pte_page(pkmap_page_table[i]);
        pte_clear(&init_mm, (unsigned long)page_address(page),
              &pkmap_page_table[i]);

        set_page_address(page, NULL);
        need_flush = 1;
    }
    if (need_flush)
        flush_tlb_kernel_range(PKMAP_ADDR(0), PKMAP_ADDR(LAST_PKMAP));
}
```

对于计数器，值为 `1` 代表表项空闲，但相应的 TLB 表项还未刷新，因此不能使用；大于 `1` 才表示在被使用。

个人理解这么做的好处是，刷新 TLB 的操作被延迟到没有直接能使用的表项时才进行，而不是每次有不使用的表项就刷新，后者效率太低了。

### `kunmap()` 函数

`kmap()` 函数用于撤销 `kmap()` 建立的永久内核映射，该函数是体系结构相关的。

`arch/x86/mm/highmem_32.c`：

```c
void kunmap(struct page *page)
{
    if (in_interrupt())
        BUG();
    if (!PageHighMem(page))
        return;
    kunmap_high(page);
}
```

如果该页框不在高端内存中，直接返回；反之调用 `kunmap_high()` 函数。

`mm/highmem.c`：

```c
/**
 * kunmap_high - map a highmem page into memory
 * @page: &struct page to unmap
 *
 * If ARCH_NEEDS_KMAP_HIGH_GET is not defined then this may be called
 * only from user context.
 */
void kunmap_high(struct page *page)
{
    unsigned long vaddr;
    unsigned long nr;
    unsigned long flags;
    int need_wakeup;

    lock_kmap_any(flags);
    vaddr = (unsigned long)page_address(page);
    BUG_ON(!vaddr);
    nr = PKMAP_NR(vaddr);

    /*
     * A count must never go down to zero
     * without a TLB flush!
     */
    need_wakeup = 0;
    switch (--pkmap_count[nr]) {
    case 0:
        BUG();
    case 1:
        /*
         * Avoid an unnecessary wake_up() function call.
         * The common case is pkmap_count[] == 1, but
         * no waiters.
         * The tasks queued in the wait-queue are guarded
         * by both the lock in the wait-queue-head and by
         * the kmap_lock.  As the kmap_lock is held here,
         * no need for the wait-queue-head's lock.  Simply
         * test if the queue is empty.
         */
        need_wakeup = waitqueue_active(&pkmap_map_wait);
    }
    unlock_kmap_any(flags);

    /* do wake-up, if needed, race-free outside of the spin lock */
    if (need_wakeup)
        wake_up(&pkmap_map_wait);
}

EXPORT_SYMBOL(kunmap_high);
```

`kunmap_high()` 函数把页框相应的计数器减一。如果该计数器的值为 `1`，那么说明没有进程在使用该页了，此时可以唤醒由 `map_new_virtual()` 阻塞的进程。当然，如果队列为空，这一步唤醒就不需要了（`need_wakeup = waitqueue_active(&pkmap_map_wait)`）。

## 临时内核映射

临时内核映射比永久内核映射的实现要简单。它不会阻塞当前进程，可以用于中断处理程序和可延迟函数。但它必须保证当前没有其他的内核控制路径在使用相同的映射，为了确保同一窗口不会被两个不同的路径同时使用，每个内核成分只能使用专门提供给它的窗口。留给临时映射的窗口很少，因此只能同时建立很少的临时内核映射。

### 窗口

这部分内容是体系结构相关的，每个 CPU 都包含多个窗口的集合。

`include/asm-generic/kmap_types.h`：

```c
#ifdef __WITH_KM_FENCE
# define KMAP_D(n) __KM_FENCE_##n ,
#else
# define KMAP_D(n)
#endif

enum km_type {
KMAP_D(0)   KM_BOUNCE_READ,
KMAP_D(1)   KM_SKB_SUNRPC_DATA,
KMAP_D(2)   KM_SKB_DATA_SOFTIRQ,
KMAP_D(3)   KM_USER0,
KMAP_D(4)   KM_USER1,
KMAP_D(5)   KM_BIO_SRC_IRQ,
KMAP_D(6)   KM_BIO_DST_IRQ,
KMAP_D(7)   KM_PTE0,
KMAP_D(8)   KM_PTE1,
KMAP_D(9)   KM_IRQ0,
KMAP_D(10)  KM_IRQ1,
KMAP_D(11)  KM_SOFTIRQ0,
KMAP_D(12)  KM_SOFTIRQ1,
KMAP_D(13)  KM_SYNC_ICACHE,
KMAP_D(14)  KM_SYNC_DCACHE,
/* UML specific, for copy_*_user - used in do_op_one_page */
KMAP_D(15)  KM_UML_USERCOPY,
KMAP_D(16)  KM_IRQ_PTE,
KMAP_D(17)  KM_NMI,
KMAP_D(18)  KM_NMI_PTE,
KMAP_D(19)  KM_TYPE_NR
};
```

因为同一窗口不能被两个不同的控制路径同时使用，所以这里的每个符号都以允许使用它的内核成分来命名。`KM_TYPE_NR` 用于记录总共的窗口个数。

临时内核映射用 `kmap_atomic()` 函数建立，用 `kunmap_atomic()` 函数撤销，这两个函数又涉及一大堆其他函数的调用，具体细节我暂时不想去了解了。
