---
title: "动态调试"
date: 2020-10-15
description: ""
menu:
  sidebar:
    name: "动态调试"
    identifier: kernel-miscellaneous-dynamic-debug
    parent: kernel-miscellaneous
    weight: 100
tags: ["Linux 内核", "动态调试"]
categories: ["Kernel"]
---

根据 Linux Kernel 在线文档 [Dynamic debug](https://www.kernel.org/doc/html/latest/admin-guide/dynamic-debug-howto.html) 整理。

<!--more-->

## 简介

动态调试（dyndbg）的主要功能是允许我们动态地打开或关闭内核代码的各种提示信息，如启用 `pr_debug()` 和 `dev_debug()` 之类的函数。

注意，打开动态调试功能需要配置内核 `Kernel hacking -> printk and dmesg options -> Enable dynamic printk() support`。

## 控制调试行为

通过往 `debugfs` 文件系统上的一个控制文件写入数据，来控制 `pr_debug()` 和 `dev_debug()` 的行为。

首先，必须挂载 `debugfs` 文件系统（如果没自动挂载的话）：

```bash
mount -t debugfs none /sys/kernel/debug
```

此时，控制文件位于 `<debugfs>/dynamic_debug/control`。

例如，启用文件 `svcsock.c` 第 `1603` 行的调试内容：

```bash
echo 'file svcsock.c line 1603 +p' > <debugfs>/dynamic_debug/control
```

如果语法错误，那么写入操作将会失败：

```bash
$ echo 'file svcsock.c wtf 1 +p' > <debugfs>/dynamic_debug/control
-bash: echo: write error: Invalid argument
```

## 查看调试行为

直接访问 `<debugfs>/dynamic_debug/control` 文件即可。比如：

```bash
$ cat <debugfs>/dynamic_debug/control
# filename:lineno [module]function flags format
net/sunrpc/svc_rdma.c:323 [svcxprt_rdma]svc_rdma_cleanup =_ "SVCRDMA Module Removed, deregister RPC RDMA transport\012"
net/sunrpc/svc_rdma.c:341 [svcxprt_rdma]svc_rdma_init =_ "\011max_inline       : %d\012"
net/sunrpc/svc_rdma.c:340 [svcxprt_rdma]svc_rdma_init =_ "\011sq_depth         : %d\012"
net/sunrpc/svc_rdma.c:338 [svcxprt_rdma]svc_rdma_init =_ "\011max_requests     : %d\012"
...
```

## 命令语言语法

### 空白符

多个空白符（空格和制表符）和单个空白符是等价的。

举个例子，下面三行命令语言是等价的：

```bash
echo -n 'file svcsock.c line 1603 +p' > <debugfs>/dynamic_debug/control
echo -n '  file   svcsock.c     line  1603 +p  ' > <debugfs>/dynamic_debug/control
echo -n 'file svcsock.c line 1603 +p' > <debugfs>/dynamic_debug/control
```

### 通配符

可以使用通配符来匹配多个内容。

举个例子，匹配所有的 usb 驱动：

```bash
echo "file drivers/usb/* +p" > <debugfs>/dynamic_debug/control
```

### 关键词

```text
command ::= match-spec* flags-spec

match-spec ::= 'func' string |
               'file' string |
               'module' string |
               'format' string |
               'line' line-range

line-range ::= lineno |
               '-'lineno |
               lineno'-' |
               lineno'-'lineno

lineno ::= unsigned-int
```

注意，`line-range` 不能包含空格，比如 `1-30` 是合法的，而 `1 - 30` 是非法的。

#### `func`

将字符串与函数名称比较。

```text
func svc_tcp_accept
func *recv*             # in rfcomm, bluetooth, ping, tcp
```

#### `file`

将字符串与相对路径名或源文件名比较。

```text
file svcsock.c
file kernel/freezer.c   # ie column 1 of control file
file drivers/usb/*      # all callsites under it
file inode.c:start_*    # parse :tail as a func (above)
file inode.c:1-100      # parse :tail as a line-range (above)
```

#### `module`

将字符串与模块名进行比较。模块名是用 `lsmod` 查看到的模块名称，去掉 `.ko` 后缀，且如果名称包含连字符 `-`，转换成下划线 `_`。

```text
module sunrpc
module nfsd
module drm*     # both drm, drm_kms_helper
```

#### `format`

将字符串与动态调试用到的格式化字符串比较。字符串不是完整匹配，只需要部分匹配。特殊字符可以用 C 语言的转义字符来表示，如 `\040` 代表空格符。或者，也可以用单引号或双引号包裹字符串。

```text
format svcrdma:         // many of the NFS/RDMA server pr_debugs
format readahead        // some pr_debugs in the readahead cache
format nfsd:\040SETATTR // one way to match a format with whitespace
format "nfsd: SETATTR"  // a neater way to match a format with whitespace
format 'nfsd: SETATTR'  // yet another way to match a format with whitespace
```

#### `line`

将行数与动态调试函数的行号比较。

line 1603           // exactly line 1603
line 1600-1605      // the six lines from line 1600 to line 1605
line -1605          // the 1605 lines from line 1 to line 1605
line 1600-          // all lines from line 1600 to the end of the file

#### `flags-spec`

`flags-spec` 由一个更改操作符（change operation）和标志（flag characters）构成。

| 更改操作符 | 用途 |
| :---: | :--- |
| `-` | 移除标志 |
| `+` | 添加标志 |
| `=` | 设置标志 |

| 标志 | 用途 |
| :---: | :--- |
| `p` | 启用动态调试函数 |
| `f` | 在输出信息中包含函数名称 |
| `l` | 在输出信息中包含行号 |
| `m` | 在输出信息中包含模块名 |
| `t` | 在输出信息中包含线程号（不是中断上下文产生的线程号） |
| `_` | 不设置标志 |

对于 `print_hex_dump_debug()` 和 `print_hex_dump_bytes()`, 只有 `p` 起作用，其他标志将被忽略。

显示标志的时候，标志前面会带有 `=`（代表当前标志等于什么）。

要一次性清楚所有标志，使用 `=_` 或 `-flmpt`。

## 引导过程中的调试信息

要在引导过程中（甚至在用户空间和 `debugfs` 存在）为核心代码和内置模块激活调试消息，使用 `dyndbg="QUERY"`、`module.dyndbg="QUERY"`。 `QUERY` 遵循命令语言语法，但不得超过 `1023` 个字符，引导程序可能会施加额外的限制。

这些参数是在 `early_initcall` 中 `ddebug` 表处理完之后进行处理的。因此，此引导参数可以启用在 `early_initcall` 之后运行的所有代码中的调试消息。

如果 `foo` 模块不是内置的，则 `foo.dyndbg` 仍将在引导时进行处理，但没有任何效果，但是在以后加载模块时将对其进行重新处理。而 `dyndbg=` 只在引导时被处理。

## 模块初始化时的调试信息

有三种方式能在模块初始化时设置动态调试。

### 配置 `/etc/modprobe.d/*.conf`

```text
options foo dyndbg=+pt
options foo dyndbg # defaults to +p
```

### 设置引导参数

```text
foo.dyndbg=" func bar +p; func buz +mp"
```

### 设置 `modprobe` 参数

```bash
modprobe foo dyndbg==pmf # override previous settings
```

## 实例

### 命令语言

```bash
// enable the message at line 1603 of file svcsock.c
echo -n 'file svcsock.c line 1603 +p' > <debugfs>/dynamic_debug/control

// enable all the messages in file svcsock.c
echo -n 'file svcsock.c +p' > <debugfs>/dynamic_debug/control

// enable all the messages in the NFS server module
echo -n 'module nfsd +p' > <debugfs>/dynamic_debug/control

// enable all 12 messages in the function svc_process()
echo -n 'func svc_process +p' > <debugfs>/dynamic_debug/control

// disable all 12 messages in the function svc_process()
echo -n 'func svc_process -p' > <debugfs>/dynamic_debug/control

// enable messages for NFS calls READ, READLINK, READDIR and READDIR+.
echo -n 'format "nfsd: READ" +p' > <debugfs>/dynamic_debug/control

// enable messages in files of which the paths include string "usb"
echo -n '*usb* +p' > <debugfs>/dynamic_debug/control

// enable all messages
echo -n '+p' > <debugfs>/dynamic_debug/control

// add module, function to all enabled messages
echo -n '+mf' > <debugfs>/dynamic_debug/control
```

### 引导参数

```text
// boot-args example, with newlines and comments for readability
Kernel command line: ...
  // see whats going on in dyndbg=value processing
  dynamic_debug.verbose=1
  // enable pr_debugs in 2 builtins, #cmt is stripped
  dyndbg="module params +p #cmt ; module sys +p"
  // enable pr_debugs in 2 functions in a module loaded later
  pc87360.dyndbg="func pc87360_init_device +p; func pc87360_find +p"
```
