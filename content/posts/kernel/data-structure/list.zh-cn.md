---
title: "内核链表"
date: 2020-09-28
description: ""
menu:
  sidebar:
    name: "内核链表"
    identifier: kernel-data-structure-list
    parent: kernel-data-structure
    weight: 100
tags: ["Linux 内核", "内核数据结构", "链表"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，GCC 扩展语法和内核数据结构部分，简单介绍 Linux 内核链表。

<!--more-->

## 传统的链表与内核链表

在通常情况下，假设我们需要一个描述狐狸信息的双向链表：

```c
struct fox {
    unsigned long tail_length; /* 尾巴长度，以厘米为单位 */
    unsigned long weight;      /* 重量，以千克为单位 */
    bool is_fantastic;         /* 这只狐狸奇妙吗? */
    struct fox *next;          /* 指向下一只狐狸 */
    struct fox *prev;          /* 指向前一只狐狸 */
};
```

我们会在 `fox` 结构体添加该结构体的指针，这样 `fox` 数据结构（即链表的结点）就能被塞入链表。

然后我们针对 `fox` 结构体实现相关的链表操作，比如添加狐狸：

```c
void add_fox(struct fox *list, struct fox *f)
{
    ...
}
```

现在我们又需要一个描述兔子信息的双向链表，自然我们需要定义一个新的结构体：

```c
struct rabbit {
    unsigned long weight;      /* 重量，以千克为单位 */
    bool is_fantastic;         /* 这只兔子奇妙吗? */
    struct rabbit *next;       /* 指向下一只兔子 */
    struct rabbit *prev;       /* 指向前一只兔子 */
};
```

然后我们需要针对 `rabbit` 结构体实现相关的链表操作，比如添加狐狸：

```c
void add_rabbit(struct rabbit *list, struct rabbit *r)
{
    ...
}
```

这时候就有人提出了，我们已经为 `fox` 结构体定义过了相关的操作，但为了 `rabbit` 结构体我们需要再次实现几乎完全一样的功能，这是重复劳动。

我们希望实现一个更加通用的方案，针对链表的操作应该对所有情况都适用（比如上面的 `fox` 和 `rabbit`）。显然传统的链表是无法解决这一问题的，因而与传统的方式相反，内核把链表结点塞入其他数据结构，实现了一种独特解法。

内核链表是一个独特的双向循环链表，下面我们先展示内核链表的用法，再介绍其具体实现。

## 内核链表的使用

内核链表实现称为 `list`，头文件为 `include/linux/list.h`。它实现了通用的链表操作，仍然以 `fox` 结构体为例，下面展示内核链表的常见用法。

### 定义链表

```c
struct fox {
    unsigned long tail_length; /* 尾巴长度，以厘米为单位 */
    unsigned long weight;      /* 重量，以千克为单位 */
    bool is_fantastic;         /* 这只狐狸奇妙吗? */
    struct list_head list;     /* 所有 fox 结构体形成链表 */
};
```

### 初始化链表

#### 动态初始化

```c
struct fox *f;
f = kmalloc(sizeof(*f), GFP_KERNEL);
f->tail_length = 40;
f->weight = 6;
f->is_fantastic = false;
INIT_LIST_HEAD(&f->list);
```

#### 静态初始化

```c
struct fox f = {
    .tail_length = 40,
    .weight = 6,
    .list = LIST_HEAD_INIT(f.list),
};
```

#### 定义链表头

```c
static LIST_HEAD(fox_list);
```

### 添加结点

#### 在头部添加结点

```c
list_add(&f->list, &fox_list);
```

#### 在尾部添加结点

```c
list_add_tail(&f->list, &fox_list);
```

### 删除结点

```c
list_del(&f->list);
```

### 遍历链表

#### 基本方法

```c
struct list_head *p;
struct fox *f;
list_for_each(p, &fox_list) {
    f = list_entry(p, struct fox, list);
    ...
}
```

#### 常用方法

```c
struct fox *f;
list_for_each_entry(f, &fox_list, list) {
    ...
}
```

#### 遍历时删除

```c
struct fox *f, *next;
list_for_each_entry_safe(f, next, &fox_list, list) {
    ...
}
```

## 内核链表的实现

所有内核链表的实现均位于 `include/linux/list.h`。

### 链表结构体的实现

```c
struct list_head {
    struct list_head *next, *prev;
};
```

### 初始化链表的实现

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)

static inline void INIT_LIST_HEAD(struct list_head *list)
{
    list->next = list;
    list->prev = list;
}
```

### 相关判断的实现

```c
/**
 * list_is_last - tests whether @list is the last entry in list @head
 * @list: the entry to test
 * @head: the head of the list
 */
static inline int list_is_last(const struct list_head *list,
                               const struct list_head *head)
{
    return list->next == head;
}

/**
 * list_empty - tests whether a list is empty
 * @head: the list to test.
 */
static inline int list_empty(const struct list_head *head)
{
    return head->next == head;
}

/**
 * list_empty_careful - tests whether a list is empty and not being modified
 * @head: the list to test
 *
 * Description:
 * tests whether a list is empty _and_ checks that no other CPU might be
 * in the process of modifying either member (next or prev)
 *
 * NOTE: using list_empty_careful() without synchronization
 * can only be safe if the only activity that can happen
 * to the list entry is list_del_init(). Eg. it cannot be used
 * if another CPU could re-list_add() it.
 */
static inline int list_empty_careful(const struct list_head *head)
{
    struct list_head *next = head->next;
    return (next == head) && (next == head->prev);
}

/**
 * list_is_singular - tests whether a list has just one entry.
 * @head: the list to test.
 */
static inline int list_is_singular(const struct list_head *head)
{
    return !list_empty(head) && (head->next == head->prev);
}
```

### 添加结点的实现

```c
static inline void __list_add(struct list_head *new,
                              struct list_head *prev,
                              struct list_head *next)
{
    next->prev = new;
    new->next = next;
    new->prev = prev;
    prev->next = new;
}

/**
 * list_add - add a new entry
 * @new: new entry to be added
 * @head: list head to add it after
 *
 * Insert a new entry after the specified head.
 * This is good for implementing stacks.
 */
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);
}

/**
 * list_add_tail - add a new entry
 * @new: new entry to be added
 * @head: list head to add it before
 *
 * Insert a new entry before the specified head.
 * This is useful for implementing queues.
 */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);
}

```

### 删除结点的实现

```c
/*
 * Delete a list entry by making the prev/next entries
 * point to each other.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
    next->prev = prev;
    prev->next = next;
}

/**
 * list_del - deletes entry from list.
 * @entry: the element to delete from the list.
 * Note: list_empty() on entry does not return true after this, the entry is
 * in an undefined state.
 */
static inline void list_del(struct list_head *entry)
{
    __list_del(entry->prev, entry->next);
    entry->next = LIST_POISON1;
    entry->prev = LIST_POISON2;
}

/**
 * list_del_init - deletes entry from list and reinitialize it.
 * @entry: the element to delete from the list.
 */
static inline void list_del_init(struct list_head *entry)
{
    __list_del(entry->prev, entry->next);
    INIT_LIST_HEAD(entry);
}
```

值得注意的是，`list_del()` 函数在调用 `__list_del()` 函数完成删除 `entry` 后，仍然设置了 `entry` 的指针。`LIST_POISON1` 和 `LIST_POISON2` 被定义于 `include/linux/poison.h`：

```c
/********** include/linux/list.h **********/

/*
 * Architectures might want to move the poison pointer offset
 * into some well-recognized area such as 0xdead000000000000,
 * that is also not mappable by user-space exploits:
 */
#ifdef CONFIG_ILLEGAL_POINTER_VALUE
# define POISON_POINTER_DELTA _AC(CONFIG_ILLEGAL_POINTER_VALUE, UL)
#else
# define POISON_POINTER_DELTA 0
#endif

/*
 * These are non-NULL pointers that will result in page faults
 * under normal circumstances, used to verify that nobody uses
 * non-initialized list entries.
 */
#define LIST_POISON1 ((void *) 0x00100100 + POISON_POINTER_DELTA)
#define LIST_POISON2 ((void *) 0x00200200 + POISON_POINTER_DELTA)
```

根据注释，设置这两个值是为了在循环中遍历到 `entry` 时触发页面异常，这相当于加了一道保险，毕竟不应该出现遍历已经删除了的结点的情况。

与 `list_del()` 形成对比，`list_del_init()` 在删除 `entry` 后，重新初始化 `entry`，这是为了能再次使用包含 `entry` 的数据结构，比如需要把该数据结构重新添加到链表中（正如刚刚说的，如果使用 `list_add()` 添加 `list_del()` 后的 `entry`，遍历时会发生页面异常）。

### 移动结点的实现

```c
/**
 * list_move - delete from one list and add as another's head
 * @list: the entry to move
 * @head: the head that will precede our entry
 */
static inline void list_move(struct list_head *list, struct list_head *head)
{
    __list_del(list->prev, list->next);
    list_add(list, head);
}

/**
 * list_move_tail - delete from one list and add as another's tail
 * @list: the entry to move
 * @head: the head that will follow our entry
 */
static inline void list_move_tail(struct list_head *list,
                                  struct list_head *head)
{
    __list_del(list->prev, list->next);
    list_add_tail(list, head);
}

/**
 * list_rotate_left - rotate the list to the left
 * @head: the head of the list
 */
static inline void list_rotate_left(struct list_head *head)
{
    struct list_head *first;

    if (!list_empty(head)) {
        first = head->next;
        list_move_tail(first, head);
    }
}
```

其中，`list_rotate_left()` 函数用于将链表的第一个结点移动至末尾。

### 拆分链表的实现

```c
static inline void __list_cut_position(struct list_head *list,
                struct list_head *head, struct list_head *entry)
{
    struct list_head *new_first = entry->next;
    list->next = head->next;
    list->next->prev = list;
    list->prev = entry;
    entry->next = list;
    head->next = new_first;
    new_first->prev = head;
}

/**
 * list_cut_position - cut a list into two
 * @list: a new list to add all removed entries
 * @head: a list with entries
 * @entry: an entry within head, could be the head itself
 *      and if so we won't cut the list
 *
 * This helper moves the initial part of @head, up to and
 * including @entry, from @head to @list. You should
 * pass on @entry an element you know is on @head. @list
 * should be an empty list or a list you do not care about
 * losing its data.
 *
 */
static inline void list_cut_position(struct list_head *list,
                struct list_head *head, struct list_head *entry)
{
    if (list_empty(head))
        return;
    if (list_is_singular(head) &&
        (head->next != entry && head != entry))
        return;
    if (entry == head)
        INIT_LIST_HEAD(list);
    else
        __list_cut_position(list, head, entry);
}
```

### 合并链表的实现

```c
static inline void __list_splice(const struct list_head *list,
                                 struct list_head *prev,
                                 struct list_head *next)
{
    struct list_head *first = list->next;
    struct list_head *last = list->prev;

    first->prev = prev;
    prev->next = first;

    last->next = next;
    next->prev = last;
}

/**
 * list_splice - join two lists, this is designed for stacks
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 */
static inline void list_splice(const struct list_head *list,
                               struct list_head *head)
{
    if (!list_empty(list))
        __list_splice(list, head, head->next);
}

/**
 * list_splice_tail - join two lists, each list being a queue
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 */
static inline void list_splice_tail(struct list_head *list,
                                    struct list_head *head)
{
    if (!list_empty(list))
        __list_splice(list, head->prev, head);
}

/**
 * list_splice_init - join two lists and reinitialise the emptied list.
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 *
 * The list at @list is reinitialised
 */
static inline void list_splice_init(struct list_head *list,
                                    struct list_head *head)
{
    if (!list_empty(list)) {
        __list_splice(list, head, head->next);
        INIT_LIST_HEAD(list);
    }
}

/**
 * list_splice_tail_init - join two lists and reinitialise the emptied list
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 *
 * Each of the lists is a queue.
 * The list at @list is reinitialised
 */
static inline void list_splice_tail_init(struct list_head *list,
                                         struct list_head *head)
{
    if (!list_empty(list)) {
        __list_splice(list, head->prev, head);
        INIT_LIST_HEAD(list);
    }
}
```

### 获取包含结点的数据结构

```c
/**
 * list_entry - get the struct for this entry
 * @ptr:    the &struct list_head pointer.
 * @type:   the type of the struct this is embedded in.
 * @member: the name of the list_struct within the struct.
 */
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)

/**
 * list_first_entry - get the first element from a list
 * @ptr:    the list head to take the element from.
 * @type:   the type of the struct this is embedded in.
 * @member: the name of the list_struct within the struct.
 *
 * Note, that list is expected to be not empty.
 */
#define list_first_entry(ptr, type, member) \
    list_entry((ptr)->next, type, member)
```

`container_of()` 宏位于 `include/linux/kernel.h`：

```c
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:    the pointer to the member.
 * @type:   the type of the container struct this is embedded in.
 * @member: the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({                  \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type, member) );})
```

该宏用到的 `offsetof()` 宏位于 `include/linux/stddef.h`：

```c
#ifdef __compiler_offsetof
#define offsetof(TYPE,MEMBER) __compiler_offsetof(TYPE,MEMBER)
#else
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#endif
```

先来看宏 `offsetof()` 宏的思路。若编译器内置了相关功能，则使用内置的 `__compiler_offsetof()`，否则使用自定义的宏。`offsetof()` 的作用正如其名，获取 `TYPE` 类型结构体的成员 `MEMBER` 在该结构体中的偏移量：

1. `(TYPE *)0`：强制类型转换，把 `0` 转换成指向 `TYPE` 类型的指针。
2. `((TYPE *)0)->MEMBER`：获取 `TYPE` 类型结构体的成员 `MEMBER`。
3. `&((TYPE *)0)->MEMBER`：获取该成员的地址。
4. `(size_t) &((TYPE *)0)->MEMBER`：把地址强制类型转换为 `size_t` 类型。

编译器在计算结构体成员的地址时，采用“成员地址 = 结构体首地址 + 成员偏移量”的方式，而这里巧妙地借助了 `0` 地址，把结构体首地址设为 `0`，得到的成员地址就等于成员偏移量。注意，这里只是获取了 `MEMBER`，即编译器只需要进行一些简单的地址计算，而无需实际访问相关的地址，因此不会生成相应的访存指令，也就不会导致运行时错误。

再来看 `container_of()` 宏，有很多相似的地方，该宏获通过指向 `type` 类型结构体的成员 `member` 的指针 `ptr` 来获取该结构体本身：

1. `(type *)0`：强制类型转换，把 `0` 转换成指向 `type` 类型的指针。
2. `((type *)0)->member`：获取 `type` 类型结构体的成员 `member`。
3. `typeof( ((type *)0)->member )`：获取 `member` 的类型，记为 `T`。
4. `const typeof( ((type *)0)->member ) *__mptr = (ptr);`：定义一个 `const T *` 类型的变量 `__mptr`，并赋值为 `ptr`。
5. `(char *)__mptr`：将 `__mptr` 强制转换成 `char *` 类型（`char *` 指针加 `1` 的语义是增加 `1` 字节）。
6. `(char *)__mptr - offsetof(type, member)`：把 `__mptr` 的地址减去 `member` 在 `type` 类型结构体中的偏移量，计算结果为 `type` 类型结构体的首地址。
7. `(type *)( (char *)__mptr - offsetof(type, member) );`：把计算结果（结构体的首地址）强制转换为 `type *` 类型。

其中，`typeof` 关键字是 GCC 扩展语法，`typeof(exp) var` 能获取表达式 `exp` 的计算结果的类型，并声明该类型的变量 `var`。比如 `typeof(1 + 1) a;`，因为表达式 `1 + 1` 的结果是 `int` 类型，所以变量 `a` 也是 `int` 类型。

额外定义一个 `__mptr` 看似是多余的，这么做的理由是为了方便程序员发现错误。具体来说，如果 `ptr` 的类型和 `member` 不一致，那么在编译 `const typeof( ((type *)0)->member ) *__mptr = (ptr);` 时会产生警告，以便让程序员注意到自己犯了错误。

最后来看 `list_entry()` 和 `list_first_entry()` 宏，就很好理解了。以上文的 `fox` 结构体为例，通过一个指向 `struct fox` 成员 `list` 的指针 `p`，来获取该结构体本身：

```c
struct fox *f;
f = list_entry(p,          // 指向结构体成员的指针
               struct fox, // 结构体类型
               list        // 成员名
              );
```

### 遍历链表的实现

遍历链表的函数相当多，这里只列出几个。

#### 基本方法的实现

```c
/**
 * list_for_each - iterate over a list
 * @pos:  the &struct list_head to use as a loop cursor.
 * @head: the head for your list.
 */
#define list_for_each(pos, head) \
    for (pos = (head)->next; prefetch(pos->next), pos != (head); \
            pos = pos->next)
```

大部分的遍历都通过 `prefetch()` 来优化遍历速度，`prefetch()` 借助了 GCC 扩展语法，定义于 `include/linux/prefetch.h`：

```c
/*
    prefetch(x) attempts to pre-emptively get the memory pointed to
    by address "x" into the CPU L1 cache.
    prefetch(x) should not cause any kind of exception, prefetch(0) is
    specifically ok.

    prefetch() should be defined by the architecture, if not, the
    #define below provides a no-op define.

    There are 3 prefetch() macros:

    prefetch(x)     - prefetches the cacheline at "x" for read
    prefetchw(x)    - prefetches the cacheline at "x" for write
    spin_lock_prefetch(x) - prefetches the spinlock *x for taking

    there is also PREFETCH_STRIDE which is the architecure-prefered
    "lookahead" size for prefetching streamed operations.

*/

#ifndef ARCH_HAS_PREFETCH
#define prefetch(x) __builtin_prefetch(x)
#endif
```

使用这种方法遍历链表，往往要借助 `list_entry()` 来获取结构体，比较麻烦，因此不常用。

#### 常用方法的实现

```c
/**
 * list_for_each_entry - iterate over list of given type
 * @pos:    the type * to use as a loop cursor.
 * @head:   the head for your list.
 * @member: the name of the list_struct within the struct.
 */
#define list_for_each_entry(pos, head, member)                      \
    for (pos = list_entry((head)->next, typeof(*pos), member);      \
         prefetch(pos->member.next), &pos->member != (head);        \
         pos = list_entry(pos->member.next, typeof(*pos), member))
```

#### 反向遍历的实现

```c
/**
 * list_for_each_entry_reverse - iterate backwards over list of given type.
 * @pos:    the type * to use as a loop cursor.
 * @head:   the head for your list.
 * @member: the name of the list_struct within the struct.
 */
#define list_for_each_entry_reverse(pos, head, member)              \
    for (pos = list_entry((head)->prev, typeof(*pos), member);      \
         prefetch(pos->member.prev), &pos->member != (head);        \
         pos = list_entry(pos->member.prev, typeof(*pos), member))
```

#### 遍历时删除的实现

```c
/**
 * list_for_each_entry_safe - iterate over list of given type safe against removal of list entry
 * @pos:    the type * to use as a loop cursor.
 * @n:      another type * to use as temporary storage
 * @head:   the head for your list.
 * @member: the name of the list_struct within the struct.
 */
#define list_for_each_entry_safe(pos, n, head, member)              \
    for (pos = list_entry((head)->next, typeof(*pos), member),      \
            n = list_entry(pos->member.next, typeof(*pos), member); \
         &pos->member != (head);                                    \
         pos = n, n = list_entry(n->member.next, typeof(*n), member))
```
