---
title: "内核散列表"
date: 2020-10-15
description: ""
menu:
  sidebar:
    name: "内核散列表"
    identifier: kernel-data-structure-hlist
    parent: kernel-data-structure
    weight: 100
tags: ["Linux 内核", "内核数据结构", "散列表"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，GCC 扩展语法和内核数据结构部分，简单介绍 Linux 内核散列表。

<!--more-->

## 内核散列表的使用

内核散列表的实现称为 `hlist`，头文件为 `include/linux/list.h`。它的操作和内核链表基本一致，只是把函数名中的 `list`（`LIST`）替换成了 `hlist`（`HLIST`）。

内核散列表的使用可以参考[内核链表](/posts/kernel/data-structure/list)的用法，下面直接介绍其具体实现。

## 内核散列表的实现

内核散列表使用拉链法（链地址法）解决冲突，内核散列的实现是内核链表的修改版本，同样位于 `include/linux/list.h`。由于大部分实现和内核链表相似，代码本身也不复杂，这里只贴出部分代码，不再重复介绍。

### 散列表结构体的实现

```c
struct hlist_head {
    struct hlist_node *first;
};

struct hlist_node {
    struct hlist_node *next, **pprev;
};
```

散列表的表头和链表元素是不对称的，表头和链表只通过一个指针来关联。散列表不需要在常数时间内访问链表的末端。链表元素依然却是包含两个指针的，值得注意的是，指向前一个结点（此处的结点可能是表头，也可能是链表元素）的指针是一个二级指针。

为什么要这么做？下面是我的个人理解。

散列表通常是一个较大的数组，数组元素就是这里的 `struct hlist_head`。我们当然希望节省空间，所以表头只包含了一个指针（每个表头节省一个指针的空间，占用空间变为了原本的一半）。但这也就带来了新的问题，表头类型（`struct hlist_head`）和链表元素类型（`struct hlist_node`）不一样，链表元素没法像内核链表那样指向表头（内核链表的表头和链表元素类型都是 `struct list_head`）。不过表头和链表元素都包含一个指向后一个结点的指针，该指针是 `struct hlist_node *` 类型的。那我们不妨利用这个指针，把表头和链表元素统一起来，用 `struct hlist_node **pprev` 指向前一个结点的“指向后一个结点的指针”——对于表头，该指针是 `struct hlist_node *first`；对于链表元素，该指针是 `struct hlist_node *next`。

### 初始化结点的实现

#### 初始化表头的实现

```c
#define HLIST_HEAD_INIT { .first = NULL }
#define HLIST_HEAD(name) struct hlist_head name = { .first = NULL }
#define INIT_HLIST_HEAD(ptr) ((ptr)->first = NULL)
```

#### 初始化链表元素的实现

```c
static inline void INIT_HLIST_NODE(struct hlist_node *h)
{
    h->next = NULL;
    h->pprev = NULL;
}
```

### 相关判断的实现

```c
static inline int hlist_unhashed(const struct hlist_node *h)
{
    return !h->pprev;
}

static inline int hlist_empty(const struct hlist_head *h)
{
    return !h->first;
}
```

### 添加链表元素的实现

```c
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h)
{
    struct hlist_node *first = h->first;
    n->next = first;
    if (first)
        first->pprev = &n->next;
    h->first = n;
    n->pprev = &h->first;
}

/* next must be != NULL */
static inline void hlist_add_before(struct hlist_node *n,
                    struct hlist_node *next)
{
    n->pprev = next->pprev;
    n->next = next;
    next->pprev = &n->next;
    *(n->pprev) = n;
}

static inline void hlist_add_after(struct hlist_node *n,
                    struct hlist_node *next)
{
    next->next = n->next;
    n->next = next;
    next->pprev = &n->next;

    if(next->next)
        next->next->pprev  = &next->next;
}
```

### 删除链表元素的实现

```c
static inline void __hlist_del(struct hlist_node *n)
{
    struct hlist_node *next = n->next;
    struct hlist_node **pprev = n->pprev;
    *pprev = next;
    if (next)
        next->pprev = pprev;
}

static inline void hlist_del(struct hlist_node *n)
{
    __hlist_del(n);
    n->next = LIST_POISON1;
    n->pprev = LIST_POISON2;
}

static inline void hlist_del_init(struct hlist_node *n)
{
    if (!hlist_unhashed(n)) {
        __hlist_del(n);
        INIT_HLIST_NODE(n);
    }
}
```

### 移动链表元素的实现

```c
/*
 * Move a list from one list head to another. Fixup the pprev
 * reference of the first entry if it exists.
 */
static inline void hlist_move_list(struct hlist_head *old,
                   struct hlist_head *new)
{
    new->first = old->first;
    if (new->first)
        new->first->pprev = &new->first;
    old->first = NULL;
}
```

### 获取包含链表元素的数据结构

```c
#define hlist_entry(ptr, type, member) container_of(ptr,type,member)
```

### 遍历链表元素的实现

```c
#define hlist_for_each(pos, head) \
    for (pos = (head)->first; pos && ({ prefetch(pos->next); 1; }); \
         pos = pos->next)

#define hlist_for_each_safe(pos, n, head) \
    for (pos = (head)->first; pos && ({ n = pos->next; 1; }); \
         pos = n)

/**
 * hlist_for_each_entry - iterate over list of given type
 * @tpos:   the type * to use as a loop cursor.
 * @pos:    the &struct hlist_node to use as a loop cursor.
 * @head:   the head for your list.
 * @member: the name of the hlist_node within the struct.
 */
#define hlist_for_each_entry(tpos, pos, head, member)            \
    for (pos = (head)->first;                                    \
         pos && ({ prefetch(pos->next); 1;}) &                   \
        ({ tpos = hlist_entry(pos, typeof(*tpos), member); 1;}); \
         pos = pos->next)

/**
 * hlist_for_each_entry_safe - iterate over list of given type safe against removal of list entry
 * @tpos:   the type * to use as a loop cursor.
 * @pos:    the &struct hlist_node to use as a loop cursor.
 * @n:      another &struct hlist_node to use as temporary storage
 * @head:   the head for your list.
 * @member: the name of the hlist_node within the struct.
 */
#define hlist_for_each_entry_safe(tpos, pos, n, head, member)    \
    for (pos = (head)->first;                                    \
         pos && ({ n = pos->next; 1; }) &&                       \
        ({ tpos = hlist_entry(pos, typeof(*tpos), member); 1;}); \
         pos = n)
```
