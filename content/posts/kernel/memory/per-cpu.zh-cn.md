---
title: "per-CPU 高速缓存"
date: 2020-07-22
description: ""
menu:
  sidebar:
    name: "per-CPU 高速缓存"
    identifier: kernel-memory-per-cpu
    parent: kernel-memory
    weight: 100
tags: ["Linux 内核", "内存管理", "per-CPU 页框高速缓存"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/zh-cn/posts/kernel/kernel)，内存管理部分，简单介绍 per-CPU 高速缓存。

<!--more-->

## “hot-n-cold pages” 补丁

这部分内容好多专业名词的翻译感觉特别变扭，就直接使用英文了。该补丁是在 2.5.45 版本加入内核的，具体可以参考社区新闻 *[Hot and cold pages](https://lwn.net/Articles/14768/)*。

下面简单概括一下这篇新闻的内容。

通常认为 RAM 是保存数据最快的地方，然而真正快的是处理器（processor）板载的硬件高速缓存（cache），所以多年来，人们都在尝试优化内核的缓存行为来避免数据进到内存。新的页分配系统只是朝这个方向迈出的又一步。

内核通常知道哪些页面最近被使用过，它们大概率在硬件高速缓存中。新加入的 “hot-n-cold pages” 补丁，尝试利用这些信息，在每个内存区（zone）加入两个 per-CPU 空闲页链表。当处理器释放怀疑为 “hot” 的页面（即该页面在处理器的硬件高速缓存中）时，该页面将被推送到 hot 链表；其他则进入 cold 链表。当然，这个链表存在上界和下界。

当内核需要一页内存时，新的分配器通常会尝试从处理器的 hot 链表中获取页。即使页面将要被改写，还是使用缓存的页更好。不过有时候使用 cold 的页更有意义的，比如用于 DMA 读取操作时，此时可以使用 `GPF_COLD` 页面分配标志。

使用 per-CPU 页链表还可以减少锁的竞争。因为当页必须在 hot/cold 链表和主内存分配器之间移动时，它们以多个页的块进行传输。

注意，在此处，“处理器”的概念是广义的，即“处理器”包含 CPU，寄存器堆（register file），浮点协处理器等部件。日常生活中的“处理器”通常就是指 CPU。

## 书中介绍的“每 CPU 页框高速缓存”

《深入理解 Linux 内核》第八章“每 CPU 页框高速缓存”一节中的好多语句，我初次读的时候都没看明白。现在结合上面的新闻，再去阅读。

> 正如我们将在本在稍后看到的，内核经常请求和释放单个页框：为了提升系统性能，每个内存管理区定义了一个“每 CPU” 页框高速缓存。所有“每 CPU” 高速缓存包含一些预先分配的页框。它们被用于满足本地 CPU 发出的单一内存请求。

现在的处理器，通常是多核的，而且带有多级缓存，一级缓存是每个核私有的，二级缓存开始可能就是共享的了。“为了满足本地 CPU 发出的单一内存请求”，这就是为什么下文说的要为每个 CPU 提供两个高速缓存（两个链表）。

> 实际上，这里为每个内存管理区和每个 CPU 提供了两个高速缓存：一个热高速缓存，它存放的页框中所包含的内容很可能就在 CPU 硬件高速缓存中；还有一个冷高速缓存。

这里讲的就是 hot/cold 链表。其中，hot 链表存放的是大概率在高速缓存中的页。

> 如果内核或用户态进程在刚分配到页框后就立即向页框写，那么从热高速缓存中获得页框就对系统性能有利。实际上，每次对页框存储单元的访问将都会导致从另一个页框给硬件高速缓存“窃取”一行——当然，除非硬件高速缓存包含有一行：它映射刚被访问的“热”页框单元。

刚分配的页框通常在高速缓存中，即会被存放于 hot 链表。后面那句话有点迷，我的理解是除非高速缓存中已经有了那行数据，否则高速缓存会从内存加载那行数据，就是解释了一下高速缓存的工作方式。

> 反过来，如果页框将要被 DMA 操作填充，那么从冷高速缓存中获得页框是方便的。在这种情况下，不会涉及到 CPU，并且硬件高速缓存的行不会被修改。从冷高速缓存获得页框为其他类型的内存分配保存了热页框储备。

这里解释了在需要 DMA 操作的情况下，用 cold 链表的好处。

## per-CPU 页框高速缓存的实现

### `pageset`

`include/linux/mmzone.h`：

```c
struct per_cpu_pages {
    int count; /* number of pages in the list */
    int low; /* low watermark, refill needed */
    int high; /* high watermark, emptying needed */
    int batch; /* chunk size for buddy add/remove */
    struct list_head list; /* the list of pages */
};

struct per_cpu_pageset {
    struct per_cpu_pages pcp[2]; /* 0: hot.  1: cold */
#ifdef CONFIG_NUMA
    unsigned long numa_hit; /* allocated in intended node */
    unsigned long numa_miss; /* allocated in non intended node */
    unsigned long numa_foreign; /* was intended here, hit elsewhere */
    unsigned long interleave_hit; /* interleaver prefered this zone */
    unsigned long local_node; /* allocation from local node */
    unsigned long other_node; /* allocation from other node */
#endif
} ____cacheline_aligned_in_smp;

struct zone {
    ...
    struct per_cpu_pageset pageset[NR_CPUS];
    ...
}
```

### 通过 per-CPU 页框高速缓存分配页框

`mm/page_alloc.c`：

```c
/*
 * Obtain a specified number of elements from the buddy allocator, all under
 * a single hold of the lock, for efficiency.  Add them to the supplied list.
 * Returns the number of new pages which were placed at *list.
 */
static int rmqueue_bulk(struct zone *zone, unsigned int order,
            unsigned long count, struct list_head *list)
{
    unsigned long flags;
    int i;
    int allocated = 0;
    struct page *page;

    spin_lock_irqsave(&zone->lock, flags);
    for (i = 0; i < count; ++i) {
        page = __rmqueue(zone, order); // 从伙伴系统分配要补充的页框
        if (page == NULL)
            break;
        allocated++;
        list_add_tail(&page->lru, list); // 添加到 hot/cold 链表
    }
    spin_unlock_irqrestore(&zone->lock, flags);
    return allocated;
}

/*
 * Really, prep_compound_page() should be called from __rmqueue_bulk().  But
 * we cheat by calling it from here, in the order > 0 path.  Saves a branch
 * or two.
 */
static struct page *
buffered_rmqueue(struct zone *zone, int order, int gfp_flags)
{
    unsigned long flags;
    struct page *page = NULL;
    int cold = !!(gfp_flags & __GFP_COLD);

    if (order == 0) { // 请求一个页框
        struct per_cpu_pages *pcp;

        pcp = &zone->pageset[get_cpu()].pcp[cold]; // 获取 per_cpu_pages
        local_irq_save(flags);
        if (pcp->count <= pcp->low) // 需要添加 per-CPU 页框高速缓存
            pcp->count += rmqueue_bulk(zone, 0, // 从伙伴系统分配要补充的页框
                        pcp->batch, &pcp->list);
        if (pcp->count) { // per-CPU 页框高速缓存中存在页框，直接分配
            page = list_entry(pcp->list.next, struct page, lru);
            list_del(&page->lru); // 从 hot/cold 链表中删除
            pcp->count--;
        }
        local_irq_restore(flags);
        put_cpu();
    }

    if (page == NULL) { // 请求未满足
        spin_lock_irqsave(&zone->lock, flags);
        page = __rmqueue(zone, order); // 从伙伴系统分配请求的页框
        spin_unlock_irqrestore(&zone->lock, flags);
    }

    if (page != NULL) { // 请求得到满足
        BUG_ON(bad_range(zone, page));
        mod_page_state_zone(zone, pgalloc, 1 << order); // 修改 page_state
        prep_new_page(page, order); // 初始化页描述符

        if (gfp_flags & __GFP_ZERO) // 内存区域填充 0
            prep_zero_page(page, order, gfp_flags);

        if (order && (gfp_flags & __GFP_COMP)) // 复合页
            prep_compound_page(page, order);
    }
    return page; // 返回页描述符地址
}
```

- 部分内容涉及[伙伴系统](/zh-cn/posts/kernel/memory/buddy-system)，此处不展开了。
- 复合页：将物理上连续的两个或多个页看成一个独立的大页，具体可以参考社区新闻 *[An introduction to compound pages](https://lwn.net/Articles/619514/)*。。

### 释放页框到 per-CPU 页框高速缓存

`mm/page_alloc.c`：

```c
/*
 * Frees a list of pages.
 * Assumes all pages on list are in same zone, and of same order.
 * count is the number of pages to free, or 0 for all on the list.
 *
 * If the zone was previously in an "all pages pinned" state then look to
 * see if this freeing clears that state.
 *
 * And clear the zone's pages_scanned counter, to hold off the "all pages are
 * pinned" detection logic.
 */
static int
free_pages_bulk(struct zone *zone, int count,
        struct list_head *list, unsigned int order)
{
    unsigned long flags;
    struct page *base, *page = NULL;
    int ret = 0;

    base = zone->zone_mem_map;
    spin_lock_irqsave(&zone->lock, flags);
    zone->all_unreclaimable = 0;
    zone->pages_scanned = 0;
    while (!list_empty(list) && count--) {
        page = list_entry(list->prev, struct page, lru);
        /* have to delete it as __free_pages_bulk list manipulates */
        list_del(&page->lru); // 从 hot/cold 链表中删除
        __free_pages_bulk(page, base, zone, order); // 从伙伴系统释放页框
        ret++;
    }
    spin_unlock_irqrestore(&zone->lock, flags);
    return ret;
}

/*
 * Free a 0-order page
 */
static void FASTCALL(free_hot_cold_page(struct page *page, int cold));
static void fastcall free_hot_cold_page(struct page *page, int cold)
{
    struct zone *zone = page_zone(page); // 获取页框所在的区
    struct per_cpu_pages *pcp;
    unsigned long flags;

    arch_free_page(page, 0); // 取消映射，如果有的话

    kernel_map_pages(page, 1, 0); // 需要开启 CONFIG_DEBUG_PAGEALLOC 选项
    inc_page_state(pgfree); // 修改 page_state
    if (PageAnon(page)) // 匿名页
        page->mapping = NULL;
    free_pages_check(__FUNCTION__, page); // 检查该页的释放条件
    pcp = &zone->pageset[get_cpu()].pcp[cold]; // 获取 per_cpu_pages
    local_irq_save(flags);
    if (pcp->count >= pcp->high) // 需要移除 per-CPU 页框高速缓存
        pcp->count -= free_pages_bulk(zone, pcp->batch, &pcp->list, 0); // 从伙伴系统释放页框
    list_add(&page->lru, &pcp->list); // 添加到 hot/cold 链表
    pcp->count++;
    local_irq_restore(flags);
    put_cpu();
}

void fastcall free_hot_page(struct page *page)
{
    free_hot_cold_page(page, 0); // 释放至 hot 链表
}

void fastcall free_cold_page(struct page *page)
{
    free_hot_cold_page(page, 1); // 释放至 cold 链表
}
```

- `CONFIG_DEBUG_PAGEALLOC`：页面被释放时是整个的从内核地址空间中移除的。该选项显著地降低了速度，但它也能迅速指出特定类型的内存崩溃错误。
- 匿名页：指应用程序动态分配的堆内存，此处不做具体阐述。

在 2.6 版本的内核中，页框全部被释放至 hot 链表，但这不意味着 cold 链表是空的，当它达到下界时会通过 `buffered_rmqueue()` 添加。
