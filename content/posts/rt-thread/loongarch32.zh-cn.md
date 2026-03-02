---
title: "移植 RT-Thread 到 LoongArch32"
date: 2023-01-14
description: ""
menu:
  sidebar:
    name: "移植 RT-Thread 到 LoongArch32"
    identifier: rt-thread-loongarch32
    parent: rt-thread
    weight: 100
tags: ["RT-Thread", "LoongArch"]
categories: ["RT-Thread"]
---

简单介绍把 RT-Thread 移植到 LoongArch32 的过程。

<!--more-->

## 引言

~~离上次更新博客又是好几个月了，越来越懒了~~

RT-Thread（后文把 RT-Thread 简写为 RT）是一个小型实时操作系统，主要由 C 语言编写且易于移植。

本文后续内容都以 32 位 LoongArch 为例，并略去了一些芯片相关的实现细节，出于保密需要，完整代码不公开。

RT-Thread 的代码相对来说还是比较容易阅读的，移植的步骤可以参考 `documentation/kernel-porting/kernel-porting.md`，同时能参考一下 ARM 和 MIPS 部分芯片加入的初始提交。

## 构建系统

RT-Thread 使用的是 SCons 构建系统，该构建系统由 Python 编写。

在移植的时候，我们需要为每个目录添加一个 `SConscript` 文件，用以描述该目录下文件和子目录的编译规则。
我们还需要在 `bsp` 的芯片目录下添加一个 `SConstruct` 文件来配置一些环境信息，并添加一个 `rtconfig.py` 来调整编译工具链和编译参数等。

### 项目结构

考虑我们需要移植一个新的架构，所以我们需要在 `libcpu` 下建立一个 `loongarch` 目录，而需要适配的芯片则在 `bsp/loongson` 下建立子目录（芯片名称以 `xxx` 代替），大致的项目结构如下：

```text
rt-thread
├── bsp
│   └── loongson
│       └── xxx # 具体的芯片型号
│           ├── applications # 应用程序目录
│           │   └── SConscript
│           ├── drivers # 驱动目录
│           ├── Kconfig # 配置文件
│           ├── link.lds # 链接脚本
│           ├── README.md # 说明文档
│           ├── rtconfig.py
│           ├── SConscript
│           └── SConstruct
└── libcpu
    └── loongarch
        ├── common # 该架构通用的代码
        │   └── SConscript
        ├── xxx # 具体的处理器核
        │   └── SConscript
        └── SConscript
```

虽然严格来说，芯片相关的代码也是架构相关的，但我习惯把 `bsp` 目录下的代码称为芯片相关的代码，`libcpu` 目录下的代码称为架构相关的代码。

### 构建系统相关的代码

`bsp/loongson/xxx/SConscript` 的内容如下：

```Python
# for module compiling
import os
from building import *

cwd = GetCurrentDir()
objs = []
list = os.listdir(cwd)

for d in list:
    path = os.path.join(cwd, d)
    if os.path.isfile(os.path.join(path, 'SConscript')):
        objs = objs + SConscript(os.path.join(d, 'SConscript'))

Return('objs')
```

因为该文件只涉及文件夹，所以我们需要遍历所有子目录，将他们加入构建列表。

其他目录的 `SConscript` 的内容大致相同：

```Python
from building import *

cwd     = GetCurrentDir()
src     = Glob('*.c') + Glob('*.cpp') + Glob('*_gcc.S')
CPPPATH = [cwd]

group = DefineGroup('Applications', src, depend = [''], CPPPATH = CPPPATH)

Return('group')
```

不难理解，这是编译当前目录下所有的 `*.c`、`*.cpp` 和 `*_gcc.S` 文件（我们使用的是 GCC 编译器，因此汇编文件需要以 `_gcc.S` 结尾）。

`bsp/loongson/xxx/SConstruct` 的内容如下：

```Python
import os
import sys
import rtconfig

if os.getenv('RTT_ROOT'):
    RTT_ROOT = os.getenv('RTT_ROOT')
else:
    RTT_ROOT = os.path.join(os.getcwd(), '..', '..', '..')

sys.path = sys.path + [os.path.join(RTT_ROOT, 'tools')]
from building import *

TARGET = 'rtthread.' + rtconfig.TARGET_EXT

DefaultEnvironment(tools=[])
env = Environment(tools = ['mingw'],
    AS   = rtconfig.AS,   ASFLAGS   = rtconfig.AFLAGS,
    CC   = rtconfig.CC,   CFLAGS    = rtconfig.CFLAGS,
    CXX  = rtconfig.CXX,  CXXFLAGS  = rtconfig.CXXFLAGS,
    AR   = rtconfig.AR,   ARFLAGS   = '-rc',
    LINK = rtconfig.LINK, LINKFLAGS = rtconfig.LFLAGS)
env.PrependENVPath('PATH', rtconfig.EXEC_PATH)
env['ASCOM'] = env['ASPPCOM']

Export('RTT_ROOT')
Export('rtconfig')

# prepare building environment
objs = PrepareBuilding(env, RTT_ROOT)

# make a building
DoBuilding(TARGET, objs)
```

`bsp/loongson/xxx/rtconfig.py` 的内容如下（工具链路径略去）：

```Python
import os

# toolchains options
ARCH       = 'loongarch'
CPU        = 'xxx'
CROSS_TOOL = 'gcc'

if os.getenv('RTT_ROOT'):
    RTT_ROOT = os.getenv('RTT_ROOT')
else:
    RTT_ROOT = os.path.join(os.getcwd(), '..', '..', '..')

if os.getenv('RTT_CC'):
    CROSS_TOOL = os.getenv('RTT_CC')

# only support GNU GCC compiler
if CROSS_TOOL == 'gcc':
    PLATFORM  = 'gcc'
    EXEC_PATH = r'xxx'
else:
    print('Please make sure your toolchains is GNU GCC!')
    exit(0)

if os.getenv('RTT_EXEC_PATH'):
    EXEC_PATH = os.getenv('RTT_EXEC_PATH')

BUILD = 'debug'

if PLATFORM == 'gcc':
    # toolchains
    PREFIX     = 'loongarch32-unknown-elf-'
    CC         = PREFIX + 'gcc'
    CXX        = PREFIX + 'g++'
    AS         = PREFIX + 'gcc'
    AR         = PREFIX + 'ar'
    LINK       = PREFIX + 'gcc'
    TARGET_EXT = 'elf'
    SIZE       = PREFIX + 'size'
    OBJDUMP    = PREFIX + 'objdump'
    OBJCPY     = PREFIX + 'objcopy'

    DEVICE = ' -mabi=ilp32 -march=la32im'
    CFLAGS = DEVICE + ' -Wall -fno-delayed-branch -mno-memcpy -fno-builtin -fno-pic'
    AFLAGS = ' -c' + DEVICE + ' -x assembler-with-cpp -D__ASSEMBLY__'
    LFLAGS = DEVICE + ' -nostartfiles -Wl,--gc-sections,-Map=rtthread.map,-cref,-u,_start -T link.lds'
    CPATH  = ''
    LPATH  = ''

    if BUILD == 'debug':
        CFLAGS += ' -O0 -ggdb'
        AFLAGS += ' -ggdb'
    else:
        CFLAGS += ' -O2 -Os'

    CXXFLAGS = CFLAGS

DUMP_ACTION = OBJDUMP + ' -D -S $TARGET > rtthread.asm\n'
POST_ACTION = OBJCPY + ' -O binary $TARGET rtthread.bin\n' + SIZE + ' $TARGET \n'
```

我觉得这里的工具链路径非常奇怪，大量的芯片都是写的比较特殊路径（比如 `/opt/xxx`，甚至是某个人的家目录），这意味着 RT 的代码在另一个环境下大概率会因为找不到编译器而构建失败。
看来 RT 的代码审核是比较宽松的。
我认为写 `/usr/bin` 之类的都比上面的“特殊”路径好接受。

## `libcpu`

这部分的代码主要包括启动代码、中断/异常处理、上下文切换和栈初始化。

### 启动代码

这部分的代码和芯片高度相关，代码不能公开，主要完成以下的事情：

1. 配置一些 CSR，比如窗口配置。
2. 跳转到链接地址。
3. 清理 BSS。
4. 初始一些寄存器，比如栈指针。
5. 调用 `rtthread_startup()`。

理论上 `rtthread_startup()` 不会返回，如果返回了，说明发生了严重错误，我们可以让它死循环或者重启。

这里踩过一个坑，如果只是把代码放在 `.text` 段，最终工具链可能会把某些小函数放在起始地址 `0x8ee00000`，这样芯片执行的第一行代码就不是我们预期的。
所以这里我使用 `.section ".startup", "ax"` 来描述段，`a` 属性代表着这个段可分配的（可以是运行时加载的），这样配合链接脚本里把 `*(.startup)` 放在开头，就能确保 `_start` 后的第一行代码放在 `0x8ee00000` 的位置。

### 中断/异常处理

LoongArch 的中断向量表可以用两种方式去实现，第一种是每个中断、异常使用不同的入口，第二种则是使用同一个入口。

一开始，我参考 Linux 内核的实现，使用不同的入口，但后来我发现 RT 几乎只用到其中几个中断/异常，这样有点浪费空间，所以我改用了第二种方法。

```text
    .section ".exc_vectors", "ax"
    .extern rt_general_exc_dispatch
    .extern rt_thread_switch_interrupt_flag
    .extern rt_interrupt_from_thread
    .extern rt_interrupt_to_thread
__ebase_entry:
general_exception:
    BACKUP_T0T1
    SAVE_ALL

    move s0, sp /* save sp */

    la   sp, _system_stack /* switch to kernel stack */

    move a0, s0
    bl   rt_general_exc_dispatch

    move sp, s0 /* restore sp */

    /*
     * switch to the new thread if necessary
     */
    la   t0, rt_thread_switch_interrupt_flag
    ld.w t1, t0, 0
    beqz t1, spurious_interrupt

    st.w zero, t0, 0

    la   t0, rt_interrupt_from_thread
    ld.w t1, t0, 0
    st.w sp, t1, 0

    la   t0, rt_interrupt_to_thread
    ld.w t1, t0, 0
    ld.w sp, t1, 0

spurious_interrupt:
    RESTORE_ALL_AND_RET

    .org __ebase_entry + 0x1000
    .extern tlb_refill_handler
tlb_refill_exception:
    b    tlb_refill_handler

    .org __ebase_entry + 0x2000
    .extern machine_error_handler
machine_error_exception:
    b    machine_error_handler
```

除了必须单独处理的 TLB 重入异常和机器错误异常外，其他所有中断/异常统一通过 `rt_general_exc_dispatch()` 来处理。

```c
void rt_general_exc_dispatch(struct stack_frame *frame)
{
    rt_ubase_t estat = csrrd(LOONGARCH_CSR_ESTAT);
    rt_ubase_t exccode = (estat & CSR_ESTAT_EXC) >> CSR_ESTAT_EXC_SHIFT;
    if (exccode == 0)
    {
        rt_uint32_t mask = csrrd(LOONGARCH_CSR_ECFG) & CSR_ECFG_IM;
        rt_uint32_t pending = estat & CSR_ESTAT_IS & mask;
        if (pending & ESTAT_IS_IPI)
            rt_do_loongarch_cpu_irq(12);
        if (pending & ESTAT_IS_TIMER)
            rt_do_loongarch_cpu_irq(11);
        if (pending & ESTAT_IS_PMC)
            rt_do_loongarch_cpu_irq(10);
        if (pending & ESTAT_IS_IP7)
            rt_do_loongarch_cpu_irq(9);
        if (pending & ESTAT_IS_IP6)
            rt_do_loongarch_cpu_irq(8);
        if (pending & ESTAT_IS_IP5)
            rt_do_loongarch_cpu_irq(7);
        if (pending & ESTAT_IS_IP4)
            rt_do_loongarch_cpu_irq(6);
        if (pending & ESTAT_IS_IP3)
            rt_do_loongarch_cpu_irq(5);
        if (pending & ESTAT_IS_IP2)
            rt_do_loongarch_cpu_irq(4);
        if (pending & ESTAT_IS_IP1)
            rt_do_loongarch_cpu_irq(3);
        if (pending & ESTAT_IS_IP0)
            rt_do_loongarch_cpu_irq(2);
        if (pending & ESTAT_IS_SIP1)
            rt_do_loongarch_cpu_irq(1);
        if (pending & ESTAT_IS_SIP0)
            rt_do_loongarch_cpu_irq(0);
    }
    else if (exccode < RT_MAX_EXC)
    {
        sys_exception_handlers[exccode](frame);
    }
}
```

细节不再展开。

这里 `/* switch to the new thread if necessary */` 后面的代码可能比较费解，我放在下一节介绍。

### 上下文切换

上下文切换主要需要实现三个函数：

- `rt_hw_context_switch_interrupt()`
- `rt_hw_context_switch_to()`
- `rt_hw_context_switch()`

ARM Cortex-M 架构有一个专门的 PendSV 异常用来进行上下文切换。
显然，LoongArch 架构不存在等效的东西。

一开始，我仿照 ARM 的上下文切换方式，使用优先级最低的 SWI0 中断来实现，但后来我发现 MIPS 有个更简单粗暴的处理方式：

- 对于不在中断中的上下文切换，直接进行切换。
- 对于在中断中的上下文切换，在 `rt_hw_context_switch_interrupt()` 中标记是否需要切换，然后在中断处理函数最后统一进行切换。

具体的代码如下：

```text
/*
 * void rt_hw_context_switch_to(rt_uint32 to)
 */
    .globl rt_hw_context_switch_to
rt_hw_context_switch_to:
    ld.w    sp, a0, 0 /* get new task stack pointer */
    RESTORE_ALL_AND_RET

/*
 * void rt_hw_context_switch(rt_uint32 from, rt_uint32 to)
 */
    .globl rt_hw_context_switch
rt_hw_context_switch:
    BACKUP_T0T1
    move   t0, ra
    csrwr  t0, LOONGARCH_CSR_ERA
    SAVE_ALL

    st.w   sp, a0, 0
    ld.w   sp, a1, 0

    RESTORE_ALL_AND_RET

    .globl rt_hw_context_switch_interrupt
    .extern rt_thread_switch_interrupt_flag
    .extern rt_interrupt_from_thread
    .extern rt_interrupt_to_thread
rt_hw_context_switch_interrupt:
    la     t0, rt_thread_switch_interrupt_flag
    ld.w   t1, t0, 0
    bnez   t1, _reswitch

    li     t1, 1
    st.w   t1, t0, 0

    la     t0, rt_interrupt_from_thread
    st.w   a0, t0, 0

_reswitch:
    la     t0, rt_interrupt_to_thread
    st.w   a1, t0, 0

    jr     ra
```

现在，上一节提到的 `/* switch to the new thread if necessary */` 后面的代码就好理解了。

### 栈初始化

这里的栈初始化是针对新创建的线程的。

我们注意到，启动后整个 RT 都是跑在关中断的状态下的。
那什么时候开中断呢？
当我们创建新线程的时候，给新线程打开中断就是一个不错的选择。

```c
rt_uint8_t *rt_hw_stack_init(void *tentry, void *parameter, rt_uint8_t *stack_addr, void *texit)
{
    rt_uint8_t *stk;
    struct stack_frame *frame;
    rt_uint32_t i;

    ...

    /* Get stk aligned */
    stk = (rt_uint8_t *)RT_ALIGN_DOWN((rt_uint32_t)stack_addr, 8);
    stk -= sizeof(struct stack_frame);

    frame = (struct stack_frame *)stk;

    /* Fill stk register numbers */
    for (i = 0; i < 32; ++i)
        frame->regs[i] = 0x0;

    frame->regs[1] = (rt_ubase_t)texit;
    frame->regs[3] = (rt_ubase_t)stk;
    frame->regs[4] = (rt_ubase_t)parameter;
    frame->csr_era = (rt_ubase_t)tentry;
    frame->csr_prmd = csrrd(LOONGARCH_CSR_PRMD) | CSR_PRMD_PIE; /* enable interrupt */

    return stk;
}
```

## `bsp`

这部分的代码主要包括驱动、应用程序、配置和链接脚本。

### 驱动

目前我只实现了三个驱动：定时器、I2C 和 UART。
我把定时器（Timer）放在了驱动里实现，但部分架构是把它放在 `libcpu` 下的。

#### 板级初始化

我们需要实现一个板级初始化函数 `rt_hw_board_init()`：

```c
void rt_hw_board_init(void)
{
    rt_hw_interrupt_init();

#ifdef RT_USING_COMPONENTS_INIT
    rt_components_board_init();
#endif

#ifdef RT_USING_CONSOLE
    rt_console_set_device(RT_CONSOLE_DEVICE_NAME);
#endif

#ifdef RT_USING_HEAP
    rt_system_heap_init((void *)RT_HW_HEAP_BEGIN, (void *)RT_HW_HEAP_END);
#endif
}
```

中断/异常的初始化通常都放在首位。

紧跟着中断/异常初始化后，我选择了初始化各种组件。
这些组件就是用 `INIT_BOARD_EXPORT()` 导出的驱动。
值得注意的是，这里包含了 UART 驱动。

之后，我设置了控制台，这样结合之前的 UART 驱动，串口就能正常输出了，尽早地开启输出能方便我调试。

最后，初始化其他的东西，这里就剩堆需要初始化了。

#### 定时器

定时器的实现比较简单：

```c
#include <rtthread.h>
#include <rthw.h>

#include "board.h"
#include "interrupt.h"
#include "drv_timer.h"

static void rt_hw_timer_handler(int vector, void *param)
{
    rt_interrupt_enter();

    /* clear interrupt */
    csrwr(CSR_TINTCLR_TI, LOONGARCH_CSR_TINTCLR);

    /* increase a OS tick */
    rt_tick_increase();

    rt_interrupt_leave();
}

int rt_hw_timer_init(void)
{
    rt_ubase_t period;
    rt_ubase_t timer_config;

    period = CPU_HZ / RT_TICK_PER_SECOND;
    timer_config = period & CSR_TCFG_VAL;
    timer_config |= (CSR_TCFG_PERIOD | CSR_TCFG_EN);
    csrwr(timer_config, LOONGARCH_CSR_TCFG);

    rt_hw_interrupt_install(ESTAT_IS_TIMER_SHIFT, rt_hw_timer_handler, RT_NULL, "tick");
    rt_hw_interrupt_umask(ESTAT_IS_TIMER_SHIFT);

    return 0;
}
INIT_BOARD_EXPORT(rt_hw_timer_init);
```

#### I2C 驱动

I2C 驱动的代码如下：

```c
#include <rtthread.h>
#include <rtdevice.h>
#include <rthw.h>

#ifdef BSP_USING_I2C

#include "board.h"
#include "drv_i2c.h"

#define PRER_LO_REG  0x0
#define PRER_HI_REG  0x1
#define CTR_REG      0x2
#define TXR_REG      0x3
#define RXR_REG      0x3
#define CR_REG       0x4
#define SR_REG       0x4
#define SLV_CTRL_REG 0x7

#define CR_START     0x80
#define CR_STOP      0x40
#define CR_READ      0x20
#define CR_WRITE     0x10
#define CR_ACK       0x8
#define CR_IACK      0x1

#define SR_NOACK     0x80
#define SR_BUSY      0x40
#define SR_AL        0x20
#define SR_TIP       0x2

struct rt_i2c_device
{
    rt_ubase_t base;
    struct rt_i2c_bus_device i2c_bus;
};

static void i2c_init(struct rt_i2c_device *i2c)
{
    rt_uint8_t val;

    val = HWREG8(i2c->base + CTR_REG);
    val &= ~(1 << 7);
    HWREG8(i2c->base + CTR_REG) = val;

    HWREG8(i2c->base + PRER_LO_REG) = 0x53;
    HWREG8(i2c->base + PRER_HI_REG) = 0x4;

    val = HWREG8(i2c->base + CTR_REG);
    val |= 1 << 7;
    HWREG8(i2c->base + CTR_REG) = val;
}

static void i2c_stop(struct rt_i2c_device *i2c)
{
    do
    {
        HWREG8(i2c->base + CR_REG) = CR_STOP;
    } while (HWREG8(i2c->base + SR_REG) & SR_BUSY);
}

static rt_bool_t i2c_send_addr(struct rt_i2c_device *i2c, rt_uint8_t addr)
{
    HWREG8(i2c->base + TXR_REG) = addr;
    HWREG8(i2c->base + CR_REG) = CR_START | CR_WRITE;

    while ((HWREG8(i2c->base + SR_REG) & SR_TIP));

    if (HWREG8(i2c->base + SR_REG) & SR_NOACK)
        return RT_FALSE;

    return RT_TRUE;
}

static rt_bool_t i2c_read(struct rt_i2c_device *i2c,
                          rt_uint8_t addr, rt_uint8_t *buf, int len)
{
    int i;

    if (!i2c_send_addr(i2c, addr))
        return RT_FALSE;

    for (i = 0; i < len; i ++)
    {
        if (i == len - 1)
            HWREG8(i2c->base + CR_REG) = CR_READ | CR_ACK;
        else
            HWREG8(i2c->base + CR_REG) = CR_READ;

        while (HWREG8(i2c->base + SR_REG) & SR_TIP);

        buf[i] = HWREG8(i2c->base + RXR_REG);
    }

    return RT_TRUE;
}

static rt_bool_t i2c_write(struct rt_i2c_device *i2c,
                           rt_uint8_t addr, rt_uint8_t *buf, int len)
{
    int i;

    if (!i2c_send_addr(i2c, addr))
        return RT_FALSE;

    for (i = 0; i < len; i++)
    {
        HWREG8(i2c->base + TXR_REG) = buf[i];
        HWREG8(i2c->base + CR_REG) = CR_WRITE;

        while (HWREG8(i2c->base + SR_REG) & SR_TIP);

        if (HWREG8(i2c->base + SR_REG) & SR_NOACK)
            return RT_FALSE;
    }

    return RT_TRUE;
}

static rt_size_t i2c_master_xfer(struct rt_i2c_bus_device *bus,
                                 struct rt_i2c_msg msgs[], rt_uint32_t num)
{
    int i;
    rt_size_t count = 0;
    struct rt_i2c_msg *msg;
    struct rt_i2c_device *i2c = rt_container_of(bus, struct rt_i2c_device, i2c_bus);

    for (i = 0; i < num; i++)
    {
        msg = &msgs[i];
        if (msg->flags == RT_I2C_RD && i2c_read(i2c, msg->addr, msg->buf, msg->len))
            count++;
        else if (msg->flags == RT_I2C_WR && i2c_write(i2c, msg->addr, msg->buf, msg->len))
            count++;
    }
    i2c_stop(i2c);

    return count;
}

static const struct rt_i2c_bus_device_ops i2c_ops =
{
    .master_xfer = i2c_master_xfer,
    .slave_xfer = RT_NULL,
    .i2c_bus_control = RT_NULL
};

#ifdef RT_USING_I2C0
static struct rt_i2c_device i2c0_device =
{
    .base = LS3_I2C0_BASE
};
#endif /* RT_USING_I2C0 */

#ifdef RT_USING_I2C1
static struct rt_i2c_device i2c1_device =
{
    .base = LS3_I2C1_BASE
};
#endif /* RT_USING_I2C1 */

int rt_hw_i2c_init(void)
{
    struct rt_i2c_device *i2c;

#ifdef RT_USING_I2C0
    i2c = &i2c0_device;

    i2c->i2c_bus.ops = &i2c_ops;
    i2c_init(i2c);

    /* register I2C0 device */
    rt_i2c_bus_device_register(&i2c->i2c_bus, "i2c0");
#endif /* RT_USING_I2C0 */

#ifdef RT_USING_I2C1
    i2c = &i2c1_device;

    i2c->i2c_bus.ops = &i2c_ops;
    i2c_init(i2c);

    /* register I2C1 device */
    rt_i2c_bus_device_register(&i2c->i2c_bus, "i2c1");
#endif /* RT_USING_I2C1 */

    return 0;
}
INIT_BOARD_EXPORT(rt_hw_i2c_init);

#endif /* BSP_USING_I2C */
```

#### UART 驱动

UART 驱动的代码如下：

```c
#include <rtthread.h>
#include <rtdevice.h>
#include <rthw.h>

#ifdef BSP_USING_UART

#include "board.h"
#include "drv_uart.h"

/* UART registers */
#define UART_DAT(base)  HWREG8(base + 0x00)
#define UART_IER(base)  HWREG8(base + 0x01)
#define UART_IIR(base)  HWREG8(base + 0x02)
#define UART_FCR(base)  HWREG8(base + 0x02)
#define UART_LCR(base)  HWREG8(base + 0x03)
#define UART_MCR(base)  HWREG8(base + 0x04)
#define UART_LSR(base)  HWREG8(base + 0x05)
#define UART_MSR(base)  HWREG8(base + 0x06)

#define UART_LSB(base)  HWREG8(base + 0x00)
#define UART_MSB(base)  HWREG8(base + 0x01)

/* Interrupt enable registers */
#define IER_IRxE        0x1 /* receive valid data interrupt enable */
#define IER_ITxE        0x2 /* transmit register empty interrupt enable */
#define IER_ILE         0x4 /* receive line status interrupt enable */
#define IER_IME         0x8 /* modem status interrupt enable */

/* Interrupt identification registers */
#define IIR_IMASK       0xf /* mask */
#define IIR_RXTOUT      0xc /* receive timeout */
#define IIR_RLS         0x6 /* receive line status */
#define IIR_RXRDY       0x4 /* receive ready */
#define IIR_TXRDY       0x2 /* transmit ready */
#define IIR_NOPEND      0x1 /* nothing */
#define IIR_MLSC        0x0 /* modem status */
#define IIR_FIFO_MASK   0xc0 /* set if FIFOs are enabled */

/* fifo control register */
#define FIFO_ENABLE     0x01 /* enable fifo */
#define FIFO_RCV_RST    0x02 /* reset receive fifo */
#define FIFO_XMT_RST    0x04 /* reset transmit fifo */
#define FIFO_DMA_MODE   0x08 /* enable dma mode */
#define FIFO_TRIGGER_1  0x00 /* trigger at 1 char */
#define FIFO_TRIGGER_4  0x40 /* trigger at 4 chars */
#define FIFO_TRIGGER_8  0x80 /* trigger at 8 chars */
#define FIFO_TRIGGER_14 0xc0 /* trigger at 14 chars */

/* character format control register */
#define CFCR_DLAB       0x80 /* divisor latch */
#define CFCR_SBREAK     0x40 /* send break */
#define CFCR_PZERO      0x30 /* zero parity */
#define CFCR_PONE       0x20 /* one parity */
#define CFCR_PEVEN      0x10 /* even parity */
#define CFCR_PODD       0x00 /* odd parity */
#define CFCR_PENAB      0x08 /* parity enable */
#define CFCR_STOPB      0x04 /* 2 stop bits */
#define CFCR_8BITS      0x03 /* 8 data bits */
#define CFCR_7BITS      0x02 /* 7 data bits */
#define CFCR_6BITS      0x01 /* 6 data bits */
#define CFCR_5BITS      0x00 /* 5 data bits */

/* modem control register */
#define MCR_LOOPBACK    0x10 /* loopback */
#define MCR_IENABLE     0x08 /* output 2 = int enable */
#define MCR_DRS         0x04 /* output 1 = xxx */
#define MCR_RTS         0x02 /* enable RTS */
#define MCR_DTR         0x01 /* enable DTR */

/* line status register */
#define LSR_RCV_FIFO    0x80 /* error in receive fifo */
#define LSR_TSRE        0x40 /* transmitter empty */
#define LSR_TXRDY       0x20 /* transmitter ready */
#define LSR_BI          0x10 /* break detected */
#define LSR_FE          0x08 /* framing error */
#define LSR_PE          0x04 /* parity error */
#define LSR_OE          0x02 /* overrun error */
#define LSR_RXRDY       0x01 /* receiver ready */
#define LSR_RCV_MASK    0x1f


/* UART interrupt enable register value */
#define UARTIER_IME     (1 << 3)
#define UARTIER_ILE     (1 << 2)
#define UARTIER_ITXE    (1 << 1)
#define UARTIER_IRXE    (1 << 0)

/* UART line control register value */
#define UARTLCR_DLAB    (1 << 7)
#define UARTLCR_BCB     (1 << 6)
#define UARTLCR_SPB     (1 << 5)
#define UARTLCR_EPS     (1 << 4)
#define UARTLCR_PE      (1 << 3)
#define UARTLCR_SB      (1 << 2)

/* UART line status register value */
#define UARTLSR_ERROR   (1 << 7)
#define UARTLSR_TE      (1 << 6)
#define UARTLSR_TFE     (1 << 5)
#define UARTLSR_BI      (1 << 4)
#define UARTLSR_FE      (1 << 3)
#define UARTLSR_PE      (1 << 2)
#define UARTLSR_OE      (1 << 1)
#define UARTLSR_DR      (1 << 0)

struct rt_uart_device
{
    rt_ubase_t base;
};

static rt_err_t uart_configure(struct rt_serial_device *serial, struct serial_configure *cfg)
{
    struct rt_uart_device *uart = RT_NULL;

    RT_ASSERT(serial != RT_NULL);
    RT_ASSERT(cfg != RT_NULL);

    uart = (struct rt_uart_device *)serial->parent.user_data;

    UART_IER(uart->base) = 0; /* clear interrupt */
    UART_FCR(uart->base) = 0xc1; /* reset UART Rx/Tx */
    /* set databits, stopbits and parity. (8-bit data, 1 stopbit, no parity) */
    UART_LCR(uart->base) = 0x3;
    UART_MCR(uart->base) = 0x3;
    UART_LSR(uart->base) = 0x60;
    UART_MSR(uart->base) = 0xb0;

    return RT_EOK;
}

static rt_err_t uart_control(struct rt_serial_device *serial, int cmd, void *arg)
{
    return RT_EOK;
}

static int uart_putc(struct rt_serial_device *serial, char c)
{
    rt_uint32_t status;
    struct rt_uart_device *uart = RT_NULL;

    RT_ASSERT(serial != RT_NULL);

    uart = (struct rt_uart_device *)serial->parent.user_data;

    status = UART_LSR(uart->base);
    while (!(status & (UARTLSR_TE | UARTLSR_TFE)))
        status = UART_LSR(uart->base);

    UART_DAT(uart->base) = c;

    return 1;
}

static int uart_getc(struct rt_serial_device *serial)
{
    struct rt_uart_device *uart = RT_NULL;

    RT_ASSERT(serial != RT_NULL);

    uart = (struct rt_uart_device *)serial->parent.user_data;

    if (LSR_RXRDY & UART_LSR(uart->base))
        return UART_DAT(uart->base);

    return -1;
}

static const struct rt_uart_ops uart_ops =
{
    .configure    = uart_configure,
    .control      = uart_control,
    .putc         = uart_putc,
    .getc         = uart_getc,
    .dma_transmit = RT_NULL
};

#ifdef RT_USING_UART0
static struct rt_uart_device uart0_device =
{
    .base = LS3_UART0_BASE
};
static struct rt_serial_device serial0;
#endif /* RT_USING_UART0 */

#ifdef RT_USING_UART1
static struct rt_uart_device uart1_device =
{
    .base = LS3_UART1_BASE
};
static struct rt_serial_device serial1;
#endif /* RT_USING_UART1 */

int rt_hw_uart_init(void)
{
    struct rt_uart_device *uart;
    struct serial_configure config = RT_SERIAL_CONFIG_DEFAULT;

#ifdef RT_USING_UART0
    uart = &uart0_device;

    serial0.ops = &uart_ops;
    serial0.config = config;

    /* register UART0 device */
    rt_hw_serial_register(&serial0, "uart0", RT_DEVICE_FLAG_RDWR, uart);
#endif /* RT_USING_UART0 */

#ifdef RT_USING_UART1
    uart = &uart1_device;

    serial1.ops = &uart_ops;
    serial1.config = config;

    /* register UART1 device */
    rt_hw_serial_register(&serial1, "uart1", RT_DEVICE_FLAG_RDWR, uart);
#endif /* RT_USING_UART1 */

    return 0;
}
INIT_BOARD_EXPORT(rt_hw_uart_init);

#endif /* BSP_USING_UART */
```

### 应用程序

这里的应用程序是内核初始化完成后，调度的第一个程序。
最简单的应用程序就是一个空的 `main()`：

```c
int main(void)
{
    return 0;
}
```

虽然是应用程序，但实际上 RT 根本没有用户态，整个都是跑在内核态的。

### 配置

SCons 构建系统和内核的 KBuild 一样，使用的是 Kconfig 来进行配置的。

我们的 `Kconfig` 文件内容如下：

```text
mainmenu "RT-Thread Project Configuration"

config BSP_DIR
    string
    option env="BSP_ROOT"
    default "."

config RTT_DIR
    string
    option env="RTT_ROOT"
    default "../../../"

config PKGS_DIR
    string
    option env="PKGS_ROOT"
    default "packages"

source "$RTT_DIR/Kconfig"
source "$PKGS_DIR/Kconfig"

config SOC_XXX
    bool
    select ARCH_LOONGARCH
    select RT_USING_COMPONENTS_INIT
    select RT_USING_USER_MAIN
    default y

source "$BSP_DIR/drivers/Kconfig"
```

### 链接脚本

链接脚本主要需要指定起始地址，数据段、代码段地址，需要导出的符号、调试信息等等。
链接脚本的名字可以自定义，但需要和 `rtconfig.py` 中保持一致。

我们的 `link.lds` 内容如下：

```text
OUTPUT_FORMAT("elf32-loongarch", "elf32-loongarch",
              "elf32-loongarch")
OUTPUT_ARCH(loongarch)
ENTRY(_start)
SECTIONS
{
    . = 0x8ee00000;
    .text :
    {
        *(.startup)

        . = ALIGN(0x1000);
        __ebase_entry = .;
        KEEP(*(.exc_vectors))
        __ebase_end = .;

        . = ALIGN(4);
        *(.text)
        *(.text.*)

        *(.rodata)
        *(.rodata.*)
        *(.rodata1)
        *(.rodata1.*)

        ...
    }

    .eh_frame_hdr :
    {
         *(.eh_frame_hdr)
         *(.eh_frame_entry)
    }
    .eh_frame : ONLY_IF_RO { KEEP (*(.eh_frame)) }

    . = 0x8ee40000;
    . = ALIGN(4);
    .data :
    {
         *(.data)
         *(.data.*)

         *(.data1)
         *(.data1.*)

        ...
    }

    .stack :
    {
        . = ALIGN(8);
        _system_stack_start = .;
        . = . + 0x1000;
        _system_stack = .;
    }

    .sbss :
    {
        __bss_start = .;
        *(.sbss)
        *(.sbss.*)
        *(.dynsbss)
        *(.scommon)
    }

    .bss :
    {
        *(.bss)
        *(.bss.*)
        *(.dynbss)
        *(COMMON)
        __bss_end = .;
    }
    _end = .;

    /* Stabs debugging sections.  */
    ...
}
```

### 小结

正如官方文档所介绍的那样，RT 确实易于移植，移植一个新架构也就一千多行的代码量（如果把各种驱动都完善的话可能也就两三千行）。

目前官方的工具链支持还不完善（32 位工具链和 newlib 库没公开，可能也没有移植到高版本）。
不过 64 位的工具链和 QEMU 已经比较完善了，就差个 newlib 库没人移植。
后续有空可以整个在 QEMU 上运行的 LoongArch64 的 RT，这样就方便开源了。
但还需要移植 newlib（乍一看这个库的移植好像不复杂？），然后自己做一套工具链，这个坑有点大，不知道以后会不会填（逃

**坑已填，见[移植 RT-Thread 到 LoongArch64](/zh-cn/posts/rt-thread/loongarch64)。**
