---
title: "内核红黑树"
date: 2020-11-04
description: ""
menu:
  sidebar:
    name: "内核红黑树"
    identifier: kernel-data-structure-rbtree
    parent: kernel-data-structure
    weight: 100
tags: ["Linux 内核", "内核数据结构", "红黑树"]
categories: ["Kernel"]
---

[Linux 内核学习笔记系列](/posts/kernel/kernel)，GCC 扩展语法和内核数据结构部分，简单介绍 Linux 内核红黑树。

<!--more-->

## 红黑树简介

红黑树是一种着色的自平衡的二叉搜索树，Linux 主要的平衡二叉树数据结构就是红黑树。

红黑树满足以下几个条件：

1. 每一个结点或者着成红色，或者着成黑色。
2. 根结点和叶结点是黑色的。
3. 如果一个结点是红色的，那么它的子结点必须是黑色的。
4. 从一个结点到一个叶结点的每一条路径必须包含相同数目的黑色结点。

以上条件保证了红黑树最深的叶结点深度不会大于两倍的最浅叶结点深度（最长路径不会超过最短路径的两倍）。注意，叶结点不存储数据，都是 `NULL` 指针。

红黑树的优点是插入、删除、搜索元素都可以在对数时间内完成，缺点是实现复杂。

显然，红黑树完整的定义和性质的证明，不属于本系列学习笔记的范畴，可以参考数据结构书。

## 内核红黑树的使用

内核红黑树的实现称为 `rbtree`，头文件为 `include/linux/rbtree.h`。内核红黑树的搜索和插入操作是没有给出完整实现的，一方面是出于 C 语言不方便泛型编程，另一方面是内核开发者相信最有效的搜索和插入方法需要由用户自行实现。用户可以使用提供的辅助函数，但需要自行实现比较操作算子。

### 红黑树的创建

创建一个红黑树，需要为其分配一个根结点，并将其初始化为 `RB_ROOT`：

```c
struct rb_root root = RB_ROOT;
```

### 红黑树的搜索

下面是一个搜索的例子，`rb_search_page_cache` 函数实现了在页高速缓存中搜索一个文件区（由一个 i 结点和一个偏移量共同描述）。每个 i 结点都有自己的 `rbtree`，以关联在文件中的页偏移。该函数将搜索给定 i 结点的 `rbtree`，以寻找匹配的偏移值：

```c
static inline struct page *rb_search_page_cache(struct inode * inode,
                                                 unsigned long offset)
{
    struct rb_node *n = inode->i_rb_page_cache.rb_node;
    struct page *page;

    while (n)
    {
        page = rb_entry(n, struct page, rb_page_cache);

        if (offset < page->offset)
            n = n->rb_left;
        else if (offset > page->offset)
            n = n->rb_right;
        else
            return page;
    }
    return NULL;
}
```

辅助宏 `rb_entry` 和内核链表的 `list_entry` 相似，获取包含结点的数据结构。

查找特定的（前一个/后一个/第一个/最后一个）结点可以使用如下函数：

```c
struct rb_node *rb_next(const struct rb_node *node);
struct rb_node *rb_prev(const struct rb_node *node);
struct rb_node *rb_first(const struct rb_root *root);
struct rb_node *rb_last(const struct rb_root *root);
```

### 红黑树的插入

插入函数需要实现搜索和插入逻辑。还是以上面的情景，对应的插入函数如下：

```c
static inline struct page *__rb_insert_page_cache(struct inode *inode,
                                                   unsigned long offset,
                                                   struct rb_node *node)
{
    struct rb_node **p = &inode->i_rb_page_cache.rb_node;
    struct rb_node *parent = NULL;
    struct page *page;

    while (*p)
    {
        parent = *p;
        page = rb_entry(parent, struct page, rb_page_cache);

        if (offset < page->offset)
            p = &(*p)->rb_left;
        else if (offset > page->offset)
            p = &(*p)->rb_right;
        else
            return page;
    }

    rb_link_node(node, parent, p);

    return NULL;
}

static inline struct page *rb_insert_page_cache(struct inode *inode,
                                                unsigned long offset,
                                                struct rb_node *node)
{
    struct page *ret;
    if ((ret = __rb_insert_page_cache(inode, offset, node)))
        goto out;
    rb_insert_color(node, &inode->i_rb_page_cache);
 out:
    return ret;
}
```

辅助函数 `rb_link_node()` 负责在给定位置插入新结点，`rb_insert_color()` 负责执行复杂的再平衡操作。

### 红黑树的删除

删除结点只需要调用以下函数：

```c
void rb_erase(struct rb_node *node, struct rb_root *root);
```

### 替换单个结点

替换单个结点只需要调用以下函数，该函数不会进行再平衡操作：

```c
void rb_replace_node(struct rb_node *victim, struct rb_node *new, struct rb_root *root);
```

## 内核红黑树的实现

内核红黑树的实现均位于 `lib/rbtree.c` 和 `include/linux/rbtree.h`。本系列学习笔记的重点不在数据结构，下面给出的代码的原理可以参考数据结构书，此处不做详细介绍。个人认为通常情况下，了解红黑树基本概念和如何使用提供的函数就足够了。

### 红黑树的定义

```c
struct rb_node
{
    unsigned long  rb_parent_color;
#define RB_RED      0
#define RB_BLACK    1
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
    /* The alignment might seem pointless, but allegedly CRIS needs it */

struct rb_root
{
    struct rb_node *rb_node;
};
```

其中，`rb_parent_color` 包含了两部分，一个是指向父结点的指针，另一个是当前结点的颜色，它是 `rb_parent_color` 的低 `2` 位。因为在所有体系结构上，指针都是至少 `4` 字节对齐的，所以这么做是可行的。

正如注释所吐槽的，对齐到 `sizeof(long)` 似乎是多余的，或许这也是个历史遗留问题。

### 辅助函数/宏的实现

```c
#define rb_parent(r)       ((struct rb_node *)((r)->rb_parent_color & ~3))
#define rb_color(r)        ((r)->rb_parent_color & 1)
#define rb_is_red(r)       (!rb_color(r))
#define rb_is_black(r)     rb_color(r)
#define rb_set_red(r)      do { (r)->rb_parent_color &= ~1; } while (0)
#define rb_set_black(r)    do { (r)->rb_parent_color |= 1; } while (0)

static inline void rb_set_parent(struct rb_node *rb, struct rb_node *p)
{
    rb->rb_parent_color = (rb->rb_parent_color & 3) | (unsigned long)p;
}
static inline void rb_set_color(struct rb_node *rb, int color)
{
    rb->rb_parent_color = (rb->rb_parent_color & ~1) | color;
}

#define RB_ROOT (struct rb_root) { NULL, }
#define rb_entry(ptr, type, member) container_of(ptr, type, member)

#define RB_EMPTY_ROOT(root) ((root)->rb_node == NULL)
#define RB_EMPTY_NODE(node) (rb_parent(node) == node)
#define RB_CLEAR_NODE(node) (rb_set_parent(node, node))
```

根据字面意思就可以理解这些辅助函数/宏的用途。

### 红黑树的旋转的实现

#### 左旋

```c
static void __rb_rotate_left(struct rb_node *node, struct rb_root *root)
{
    struct rb_node *right = node->rb_right;
    struct rb_node *parent = rb_parent(node);

    if ((node->rb_right = right->rb_left))
        rb_set_parent(right->rb_left, node);
    right->rb_left = node;

    rb_set_parent(right, parent);

    if (parent)
    {
        if (node == parent->rb_left)
            parent->rb_left = right;
        else
            parent->rb_right = right;
    }
    else
        root->rb_node = right;
    rb_set_parent(node, right);
}
```

#### 右旋

```c
static void __rb_rotate_right(struct rb_node *node, struct rb_root *root)
{
    struct rb_node *left = node->rb_left;
    struct rb_node *parent = rb_parent(node);

    if ((node->rb_left = left->rb_right))
        rb_set_parent(left->rb_right, node);
    left->rb_right = node;

    rb_set_parent(left, parent);

    if (parent)
    {
        if (node == parent->rb_right)
            parent->rb_right = left;
        else
            parent->rb_left = left;
    }
    else
        root->rb_node = left;
    rb_set_parent(node, left);
}
```

### 红黑树的插入的实现

下面给出插入的辅助函数的实现。

#### 插入结点的实现

```c
static inline void rb_link_node(struct rb_node * node, struct rb_node * parent,
                struct rb_node ** rb_link)
{
    node->rb_parent_color = (unsigned long )parent;
    node->rb_left = node->rb_right = NULL;

    *rb_link = node;
}
```

#### 再平衡的实现

```c
void rb_insert_color(struct rb_node *node, struct rb_root *root)
{
    struct rb_node *parent, *gparent;

    while ((parent = rb_parent(node)) && rb_is_red(parent))
    {
        gparent = rb_parent(parent);

        if (parent == gparent->rb_left)
        {
            {
                register struct rb_node *uncle = gparent->rb_right;
                if (uncle && rb_is_red(uncle))
                {
                    rb_set_black(uncle);
                    rb_set_black(parent);
                    rb_set_red(gparent);
                    node = gparent;
                    continue;
                }
            }

            if (parent->rb_right == node)
            {
                register struct rb_node *tmp;
                __rb_rotate_left(parent, root);
                tmp = parent;
                parent = node;
                node = tmp;
            }

            rb_set_black(parent);
            rb_set_red(gparent);
            __rb_rotate_right(gparent, root);
        } else {
            {
                register struct rb_node *uncle = gparent->rb_left;
                if (uncle && rb_is_red(uncle))
                {
                    rb_set_black(uncle);
                    rb_set_black(parent);
                    rb_set_red(gparent);
                    node = gparent;
                    continue;
                }
            }

            if (parent->rb_left == node)
            {
                register struct rb_node *tmp;
                __rb_rotate_right(parent, root);
                tmp = parent;
                parent = node;
                node = tmp;
            }

            rb_set_black(parent);
            rb_set_red(gparent);
            __rb_rotate_left(gparent, root);
        }
    }

    rb_set_black(root->rb_node);
}
EXPORT_SYMBOL(rb_insert_color);
```

### 红黑树的删除的实现

```c
static void __rb_erase_color(struct rb_node *node, struct rb_node *parent,
                 struct rb_root *root)
{
    struct rb_node *other;

    while ((!node || rb_is_black(node)) && node != root->rb_node)
    {
        if (parent->rb_left == node)
        {
            other = parent->rb_right;
            if (rb_is_red(other))
            {
                rb_set_black(other);
                rb_set_red(parent);
                __rb_rotate_left(parent, root);
                other = parent->rb_right;
            }
            if ((!other->rb_left || rb_is_black(other->rb_left)) &&
                (!other->rb_right || rb_is_black(other->rb_right)))
            {
                rb_set_red(other);
                node = parent;
                parent = rb_parent(node);
            }
            else
            {
                if (!other->rb_right || rb_is_black(other->rb_right))
                {
                    rb_set_black(other->rb_left);
                    rb_set_red(other);
                    __rb_rotate_right(other, root);
                    other = parent->rb_right;
                }
                rb_set_color(other, rb_color(parent));
                rb_set_black(parent);
                rb_set_black(other->rb_right);
                __rb_rotate_left(parent, root);
                node = root->rb_node;
                break;
            }
        }
        else
        {
            other = parent->rb_left;
            if (rb_is_red(other))
            {
                rb_set_black(other);
                rb_set_red(parent);
                __rb_rotate_right(parent, root);
                other = parent->rb_left;
            }
            if ((!other->rb_left || rb_is_black(other->rb_left)) &&
                (!other->rb_right || rb_is_black(other->rb_right)))
            {
                rb_set_red(other);
                node = parent;
                parent = rb_parent(node);
            }
            else
            {
                if (!other->rb_left || rb_is_black(other->rb_left))
                {
                    rb_set_black(other->rb_right);
                    rb_set_red(other);
                    __rb_rotate_left(other, root);
                    other = parent->rb_left;
                }
                rb_set_color(other, rb_color(parent));
                rb_set_black(parent);
                rb_set_black(other->rb_left);
                __rb_rotate_right(parent, root);
                node = root->rb_node;
                break;
            }
        }
    }
    if (node)
        rb_set_black(node);
}

void rb_erase(struct rb_node *node, struct rb_root *root)
{
    struct rb_node *child, *parent;
    int color;

    if (!node->rb_left)
        child = node->rb_right;
    else if (!node->rb_right)
        child = node->rb_left;
    else
    {
        struct rb_node *old = node, *left;

        node = node->rb_right;
        while ((left = node->rb_left) != NULL)
            node = left;

        if (rb_parent(old)) {
            if (rb_parent(old)->rb_left == old)
                rb_parent(old)->rb_left = node;
            else
                rb_parent(old)->rb_right = node;
        } else
            root->rb_node = node;

        child = node->rb_right;
        parent = rb_parent(node);
        color = rb_color(node);

        if (parent == old) {
            parent = node;
        } else {
            if (child)
                rb_set_parent(child, parent);
            parent->rb_left = child;

            node->rb_right = old->rb_right;
            rb_set_parent(old->rb_right, node);
        }

        node->rb_parent_color = old->rb_parent_color;
        node->rb_left = old->rb_left;
        rb_set_parent(old->rb_left, node);

        goto color;
    }

    parent = rb_parent(node);
    color = rb_color(node);

    if (child)
        rb_set_parent(child, parent);
    if (parent)
    {
        if (parent->rb_left == node)
            parent->rb_left = child;
        else
            parent->rb_right = child;
    }
    else
        root->rb_node = child;

 color:
    if (color == RB_BLACK)
        __rb_erase_color(child, parent, root);
}
EXPORT_SYMBOL(rb_erase);
```

### 红黑树的查找的实现

下面是查找特定的结点的实现：

```c
/*
 * This function returns the first node (in sort order) of the tree.
 */
struct rb_node *rb_first(const struct rb_root *root)
{
    struct rb_node *n;

    n = root->rb_node;
    if (!n)
        return NULL;
    while (n->rb_left)
        n = n->rb_left;
    return n;
}
EXPORT_SYMBOL(rb_first);

struct rb_node *rb_last(const struct rb_root *root)
{
    struct rb_node *n;

    n = root->rb_node;
    if (!n)
        return NULL;
    while (n->rb_right)
        n = n->rb_right;
    return n;
}
EXPORT_SYMBOL(rb_last);

struct rb_node *rb_next(const struct rb_node *node)
{
    struct rb_node *parent;

    if (rb_parent(node) == node)
        return NULL;

    /* If we have a right-hand child, go down and then left as far
       as we can. */
    if (node->rb_right) {
        node = node->rb_right;
        while (node->rb_left)
            node=node->rb_left;
        return (struct rb_node *)node;
    }

    /* No right-hand children.  Everything down and left is
       smaller than us, so any 'next' node must be in the general
       direction of our parent. Go up the tree; any time the
       ancestor is a right-hand child of its parent, keep going
       up. First time it's a left-hand child of its parent, said
       parent is our 'next' node. */
    while ((parent = rb_parent(node)) && node == parent->rb_right)
        node = parent;

    return parent;
}
EXPORT_SYMBOL(rb_next);

struct rb_node *rb_prev(const struct rb_node *node)
{
    struct rb_node *parent;

    if (rb_parent(node) == node)
        return NULL;

    /* If we have a left-hand child, go down and then right as far
       as we can. */
    if (node->rb_left) {
        node = node->rb_left;
        while (node->rb_right)
            node=node->rb_right;
        return (struct rb_node *)node;
    }

    /* No left-hand children. Go up till we find an ancestor which
       is a right-hand child of its parent */
    while ((parent = rb_parent(node)) && node == parent->rb_left)
        node = parent;

    return parent;
}
EXPORT_SYMBOL(rb_prev);
```

### 替换单个结点的实现

```c
void rb_replace_node(struct rb_node *victim, struct rb_node *new,
             struct rb_root *root)
{
    struct rb_node *parent = rb_parent(victim);

    /* Set the surrounding nodes to point to the replacement */
    if (parent) {
        if (victim == parent->rb_left)
            parent->rb_left = new;
        else
            parent->rb_right = new;
    } else {
        root->rb_node = new;
    }
    if (victim->rb_left)
        rb_set_parent(victim->rb_left, new);
    if (victim->rb_right)
        rb_set_parent(victim->rb_right, new);

    /* Copy the pointers/colour from the victim to the replacement */
    *new = *victim;
}
EXPORT_SYMBOL(rb_replace_node);
```
