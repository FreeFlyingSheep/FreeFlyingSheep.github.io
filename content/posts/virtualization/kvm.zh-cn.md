---
title: "KVM 简介"
date: 2021-06-09
description: ""
menu:
  sidebar:
    name: "KVM 简介"
    identifier: virtualization-kvm
    parent: virtualization
    weight: 100
tags: ["KVM", "Linux 内核"]
categories: ["Virtualization"]
---

简单介绍 Linux 内核的 KVM 模块，以 MIPS 体系结构为例，内核源码版本为 5.10.42。

<!--more-->

## KVM (Kernel-based Virtual Machine)

KVM 是一个完整的虚拟化解决方案，允许用户在特定机器上用虚拟机直接运行未经修改的 Linux 内核。

KVM 通常和 QEMU 结合使用，见 [QEMU 简介](/zh-cn/posts/virtualization/qemu)。

KVM 的 API 接口可以参考[内核文档](https://www.kernel.org/doc/html/latest/virt/kvm/api.html)。

KVM 模块有体系结构相关和无关的两部分，简单起见，体系结构相关的部分只考虑 MIPS 的代码，选用内核源码版本为 5.10.42。

## 模块初始化

`arch/mips/kvm/mips.c`：

```c
static int __init kvm_mips_init(void)
{
    int ret;

    if (cpu_has_mmid) {
        pr_warn("KVM does not yet support MMIDs. KVM Disabled\n");
        return -EOPNOTSUPP;
    }

    ret = kvm_mips_entry_setup();
    if (ret)
        return ret;

    ret = kvm_init(NULL, sizeof(struct kvm_vcpu), 0, THIS_MODULE);

    if (ret)
        return ret;

    if (boot_cpu_type() == CPU_LOONGSON64)
        kvm_priority_to_irq = kvm_loongson3_priority_to_irq;

    register_die_notifier(&kvm_mips_csr_die_notifier);

    return 0;
}

module_init(kvm_mips_init);
```

`kvm_mips_init()` 函数会调用体系结构无关的初始化函数 `kvm_init()`。

```c
int kvm_init(void *opaque, unsigned vcpu_size, unsigned vcpu_align,
                  struct module *module)
{
    struct kvm_cpu_compat_check c;
    int r;
    int cpu;

    r = kvm_arch_init(opaque);
    if (r)
        goto out_fail;

    /*
     * kvm_arch_init makes sure there's at most one caller
     * for architectures that support multiple implementations,
     * like intel and amd on x86.
     * kvm_arch_init must be called before kvm_irqfd_init to avoid creating
     * conflicts in case kvm is already setup for another implementation.
     */
    r = kvm_irqfd_init();
    if (r)
        goto out_irqfd;

    if (!zalloc_cpumask_var(&cpus_hardware_enabled, GFP_KERNEL)) {
        r = -ENOMEM;
        goto out_free_0;
    }

    r = kvm_arch_hardware_setup(opaque);
    if (r < 0)
        goto out_free_1;

    c.ret = &r;
    c.opaque = opaque;
    for_each_online_cpu(cpu) {
        smp_call_function_single(cpu, check_processor_compat, &c, 1);
        if (r < 0)
            goto out_free_2;
    }

    r = cpuhp_setup_state_nocalls(CPUHP_AP_KVM_STARTING, "kvm/cpu:starting",
                                  kvm_starting_cpu, kvm_dying_cpu);
    if (r)
        goto out_free_2;
    register_reboot_notifier(&kvm_reboot_notifier);

    /* A kmem cache lets us meet the alignment requirements of fx_save. */
    if (!vcpu_align)
        vcpu_align = __alignof__(struct kvm_vcpu);
    kvm_vcpu_cache =
        kmem_cache_create_usercopy("kvm_vcpu", vcpu_size, vcpu_align,
                                   SLAB_ACCOUNT,
                                   offsetof(struct kvm_vcpu, arch),
                                   sizeof_field(struct kvm_vcpu, arch),
                                   NULL);
    if (!kvm_vcpu_cache) {
        r = -ENOMEM;
        goto out_free_3;
    }

    r = kvm_async_pf_init();
    if (r)
        goto out_free;

    kvm_chardev_ops.owner = module;
    kvm_vm_fops.owner = module;
    kvm_vcpu_fops.owner = module;

    r = misc_register(&kvm_dev);
    if (r) {
        pr_err("kvm: misc device register failed\n");
        goto out_unreg;
    }

    register_syscore_ops(&kvm_syscore_ops);

    kvm_preempt_ops.sched_in = kvm_sched_in;
    kvm_preempt_ops.sched_out = kvm_sched_out;

    kvm_init_debug();

    r = kvm_vfio_ops_init();
    WARN_ON(r);

    return 0;

out_unreg:
    kvm_async_pf_deinit();
out_free:
    kmem_cache_destroy(kvm_vcpu_cache);
out_free_3:
    unregister_reboot_notifier(&kvm_reboot_notifier);
    cpuhp_remove_state_nocalls(CPUHP_AP_KVM_STARTING);
out_free_2:
    kvm_arch_hardware_unsetup();
out_free_1:
    free_cpumask_var(cpus_hardware_enabled);
out_free_0:
    kvm_irqfd_exit();
out_irqfd:
    kvm_arch_exit();
out_fail:
    return r;
}
EXPORT_SYMBOL_GPL(kvm_init);
```

可以看到，`kvm_init()` 会调用 `misc_register()` 函数来注册块设备 `/dev/kvm`。

```c
static long kvm_dev_ioctl(struct file *filp,
                          unsigned int ioctl, unsigned long arg)
{
    long r = -EINVAL;

    switch (ioctl) {
    case KVM_GET_API_VERSION:
        if (arg)
            goto out;
        r = KVM_API_VERSION;
        break;
    case KVM_CREATE_VM:
        r = kvm_dev_ioctl_create_vm(arg);
        break;
    case KVM_CHECK_EXTENSION:
        r = kvm_vm_ioctl_check_extension_generic(NULL, arg);
        break;
    case KVM_GET_VCPU_MMAP_SIZE:
        if (arg)
            goto out;
        r = PAGE_SIZE;     /* struct kvm_run */
#ifdef CONFIG_X86
        r += PAGE_SIZE;    /* pio data page */
#endif
#ifdef CONFIG_KVM_MMIO
        r += PAGE_SIZE;    /* coalesced mmio ring page */
#endif
        break;
    case KVM_TRACE_ENABLE:
    case KVM_TRACE_PAUSE:
    case KVM_TRACE_DISABLE:
        r = -EOPNOTSUPP;
        break;
    default:
        return kvm_arch_dev_ioctl(filp, ioctl, arg);
    }
out:
    return r;
}

static struct file_operations kvm_chardev_ops = {
    .unlocked_ioctl = kvm_dev_ioctl,
    .llseek         = noop_llseek,
    KVM_COMPAT(kvm_dev_ioctl),
};

static struct miscdevice kvm_dev = {
    KVM_MINOR,
    "kvm",
    &kvm_chardev_ops,
};
```

## 创建虚拟机

TODO
