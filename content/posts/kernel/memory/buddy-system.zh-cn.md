---
title: "伙伴系统"
date: 2020-07-21
description: ""
menu:
  sidebar:
    name: "伙伴系统"
    identifier: kernel-memory-buddy-system
    parent: kernel-memory
    weight: 100
tags: ["Linux 内核", "内存管理", "伙伴系统"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/zh-cn/posts/kernel/kernel)，内存管理部分，简单介绍伙伴系统。

<!--more-->

## 伙伴系统简介

伙伴（Buddy）系统是为了解决外碎片问题而设计的。

伙伴系统算法把所有的空闲页框分为 11 个块链表，每个块链表分别包含 1、2、4、8、16、32、64、128、256、512 和 1024 个连续的页框。每个块的第一个页框的物理地址是该块大小的整数倍。由于内核页大小是 4 KB，所以最大请求为 4 MB 大小连续的 RAM 块。

在请求块时，算法会先在请求页框大小块的链表中检查是否有一个空闲块，如果没有，就查找下一个更大的块，把这个块分裂为两个小块。如果还是没有找到，就继续查找更大块，直到最大的 1024 个页框的块。如果 1024 个页框的链表还是空的，算法就放弃并发出错误信号。

在释放块时，算法会试图把一对空闲伙伴合并为一个单独的块，且算法是迭代的，即算法会再次试图合成的生成的块。满足以下条件的两个块被称为伙伴：

- 两个块具有相同的大小，记作 `b`。
- 它们的物理地址连续。
- 第一个块的第一个页框的物理地址是 `2 * b * 2^12`。

由于块的大小都是 2 的幂，所以被称为伙伴的块只有一位二进制位不同。对于一个块，只需要与掩码（`1 << order`）进行异或操作，就能找到它的伙伴。

## 伙伴系统分配器

### 数据结构

`include/linux/mmzone.h`：

```c
struct zone {
    ...
    /*
     * free areas of different sizes
     */
    spinlock_t lock;
    struct free_area free_area[MAX_ORDER];
    ...
    /*
     * Discontig memory support fields.
     */
    struct pglist_data *zone_pgdat;
    struct page *zone_mem_map;
    /* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
    unsigned long zone_start_pfn;

    unsigned long spanned_pages; /* total size, including holes */
    unsigned long present_pages; /* amount of memory (excluding holes) */
    ...
}
```

对每个管理区，Linux 采用不同的伙伴系统，每个伙伴系统使用的主要数据结构如下：

- `mem_map` 数组。每个区都关系到该数组的子集，由 `zone_mem_map` 指定子集的第一个元素。
- `free_area` 数组。该数组的元素类型为 `free_area`，每个元素对应一种块大小。

### 分配块

`mm/page_alloc.c`：

```c
static inline struct page *
expand(struct zone *zone, struct page *page,
    int low, int high, struct free_area *area)
{
    unsigned long size = 1 << high;

    while (high > low) { // current_order > order
        area--;
        high--;
        size >>= 1;
        BUG_ON(bad_range(zone, &page[size]));
        list_add(&page[size].lru, &area->free_list); // 添加到空闲链表
        area->nr_free++;
        set_page_order(&page[size], high); // 设置 private
    }
    return page;
}

/*
 * Do the hard work of removing an element from the buddy allocator.
 * Call me with the zone->lock already held.
 */
static struct page *__rmqueue(struct zone *zone, unsigned int order)
{
    struct free_area * area;
    unsigned int current_order;
    struct page *page;

    for (current_order = order; current_order < MAX_ORDER; ++current_order) {
        area = zone->free_area + current_order;
        if (list_empty(&area->free_list))
            continue;

        page = list_entry(area->free_list.next, struct page, lru);
        list_del(&page->lru); // 从空闲链表中删除
        rmv_page_order(page); // 清除 private
        area->nr_free--;
        zone->free_pages -= 1UL << order;
        return expand(zone, page, order, current_order, area);
    }

    return NULL;
}
```

### 释放块

`mm/page_alloc.c`：

```c
static inline void __free_pages_bulk (struct page *page, struct page *base,
        struct zone *zone, unsigned int order)
{
    unsigned long page_idx;
    struct page *coalesced;
    int order_size = 1 << order;

    if (unlikely(order)) // 复合页
        destroy_compound_page(page, order);

    page_idx = page - base; // 当前块的下标

    BUG_ON(page_idx & (order_size - 1));
    BUG_ON(bad_range(zone, page));

    zone->free_pages += order_size;
    while (order < MAX_ORDER-1) { // 合并一个块和它的伙伴
        struct free_area *area;
        struct page *buddy;
        int buddy_idx;

        buddy_idx = (page_idx ^ (1 << order)); // 当前块的伙伴的下标
        buddy = base + buddy_idx;
        if (bad_range(zone, buddy))
            break;
        if (!page_is_buddy(buddy, order))
            break;
        /* Move the buddy up one level. */
        list_del(&buddy->lru);
        area = zone->free_area + order;
        area->nr_free--;
        rmv_page_order(buddy); // 清除 private
        page_idx &= buddy_idx; // 恢复当前块的下标
        order++;
    }
    coalesced = base + page_idx;
    set_page_order(coalesced, order);
    list_add(&coalesced->lru, &zone->free_area[order].free_list); // 添加到空闲链表
    zone->free_area[order].nr_free++;
}
```

复合页：将物理上连续的两个或多个页看成一个独立的大页，具体可以参考社区新闻 *[An introduction to compound pages](https://lwn.net/Articles/619514/)*。
