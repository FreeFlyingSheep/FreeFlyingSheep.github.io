---
title: "在龙芯上移植 OpenStack"
date: 2020-11-30
description: ""
menu:
  sidebar:
    name: "在龙芯上移植 OpenStack"
    identifier: openstack-lemote
    parent: openstack
    weight: 100
tags: ["OpenStack"]
categories: ["OpenStack"]
---

整理工作中移植 OpenStack Victoria 版本核心组件到龙芯 lemote Fedora 28 系统上的过程。

<!--more-->

## 工作需求

现在需要移植 OpenStack 核心组件到 lemote Fedora 28 系统上，索性选取了当前最新的稳定分支 Victoria 版本。

移植所有组件并测试是不现实的，目前只考虑几个核心组件，包括 Nova、Cinder、Neutron、Horizo​​n、Keystone、Glance 和 Placement。

考虑到 OpenStack 是用 Python 编写的，而 Python 环境和大部分系统工具 lemote Fedora 28 系统上已经存在，而且接口是与 x86 一致的，所以不少组件应该是能直接运行的。龙芯平台的 libvirt 和 QEMU/LVM 是修改过的，因此最有可能需要修改的是 OpenStack Nova 组件的代码（事实证明，确实只需要修改这一个组件）。

为了方便部署测试，直接选择使用官方的 [DevStack](https://opendev.org/openstack/devstack) 自动部署脚本。

## 移植 DevStack

显然，DevStack 不经过修改是无法在龙芯平台上跑通的。修改过的 DevStack 代码见 <https://github.com/FreeFlyingSheep/devstack>。

### 探测版本信息

该部分代码位于 `functions-common`。

为了能在多个发行版上运行，DevStack 首先借助 `lsb_release` 命令探测发行版信息。而 lemote 系统上并没有提供 `lsb_release` 软件包，为此，通过检查 `/proc/version` 中的信息，发现是 lemote 系统直接返回，来绕过安装 `lsb_release`。

```bash
# Make a *best effort* attempt to install lsb_release packages for the
# user if not available.  Note can't use generic install_package*
# because they depend on this!
function _ensure_lsb_release {
    if [[ -x $(command -v lsb_release 2>/dev/null) ]]; then
        return
    fi

    # lemote
    if grep -q "lemote" /proc/version; then
        return
    fi

    if [[ -x $(command -v apt-get 2>/dev/null) ]]; then
        sudo apt-get install -y lsb-release
    elif [[ -x $(command -v zypper 2>/dev/null) ]]; then
        sudo zypper -n install lsb-release
    elif [[ -x $(command -v dnf 2>/dev/null) ]]; then
        sudo dnf install -y redhat-lsb-core
    else
        die $LINENO "Unable to find or auto-install lsb_release"
    fi
}
```

只是绕过安装并没解决问题，还需要手动指定 lemote 系统的发行版系统，为了和其他发行版统一，借助 `/etc/os-release` 文件来获取相关信息。

```bash
# GetOSVersion
#  Set the following variables:
#  - os_RELEASE
#  - os_CODENAME
#  - os_VENDOR
#  - os_PACKAGE
function GetOSVersion {
    # We only support distros that provide a sane lsb_release
    _ensure_lsb_release

    if ! grep -q "lemote" /proc/version; then
        os_RELEASE=$(lsb_release -r -s)
        os_CODENAME=$(lsb_release -c -s)
        os_VENDOR=$(lsb_release -i -s)
    else # lemote
        os_RELEASE=$(grep "^VERSION_ID=" /etc/os-release | cut -d '=' -f 2)
        os_CODENAME=$(grep "^VERSION=" /etc/os-release | cut -d '(' -f 2 | cut -d ')' -f 1)
        os_VENDOR=$(grep "^NAME=" /etc/os-release | cut -d '=' -f 2)
    fi

    if [[ $os_VENDOR =~ (Debian|Ubuntu|LinuxMint) ]]; then
        os_PACKAGE="deb"
    else
        os_PACKAGE="rpm"
    fi

    typeset -xr os_VENDOR
    typeset -xr os_RELEASE
    typeset -xr os_PACKAGE
    typeset -xr os_CODENAME
}
```

同时，为了方便后续判断系统信息，参考脚本提供的 `is_suse` 函数，加上 `is_lemote` 函数：

```bash
function is_lemote {
    is_fedora && grep -q "lemote" /proc/version
}
```

由于而脚本已经放弃了对 Fedora 28 系统的支持，所以运行时需要加上 `FORCE=yes`，索性将其添加到 `local.conf` 中。

### 编译 ETCD

脚本会自动根据系统版本信息来从官方下载 ETCD 包，而官方并未提供 MIPS 版本的 ETCD，所以需要手动编译。

ETCD 的编译比较简单，直接从[官方仓库](https://github.com/etcd-io/etcd)克隆，切换到相应版本（为了与脚本统一，选择了 3.3.12 版本），编译并参考其他体系结构的压缩文件打包为 `etcd-v3.3.12-linux-mips64.tar.gz`，之后在脚本中添加 MIPS 体系结构的信息。

该部分代码位于 `stackrc`。

```bash
# etcd3 defaults
ETCD_VERSION=${ETCD_VERSION:-v3.3.12}
ETCD_SHA256_AMD64=${ETCD_SHA256_AMD64:-"dc5d82df095dae0a2970e4d870b6929590689dd707ae3d33e7b86da0f7f211b6"}
ETCD_SHA256_ARM64=${ETCD_SHA256_ARM64:-"170b848ac1a071fe7d495d404a868a2c0090750b2944f8a260ef1c6125b2b4f4"}
ETCD_SHA256_PPC64=${ETCD_SHA256_PPC64:-"77f807b1b51abbf51e020bb05bdb8ce088cb58260fcd22749ea32eee710463d3"}
# etcd v3.2.x doesn't have anything for s390x
ETCD_SHA256_S390X=${ETCD_SHA256_S390X:-""}
# etcd v3.2.x doesn't have anything for mips64
ETCD_SHA256_MIPS64=${ETCD_SHA256_MIPS64:-"5632a3ce5d376701f87887ff5caac5d41a74ec6c1e691a661dc6ce1f1d2effab"}
# Make sure etcd3 downloads the correct architecture
if is_arch "x86_64"; then
    ETCD_ARCH="amd64"
    ETCD_SHA256=${ETCD_SHA256:-$ETCD_SHA256_AMD64}
elif is_arch "aarch64"; then
    ETCD_ARCH="arm64"
    ETCD_SHA256=${ETCD_SHA256:-$ETCD_SHA256_ARM64}
elif is_arch "ppc64le"; then
    ETCD_ARCH="ppc64le"
    ETCD_SHA256=${ETCD_SHA256:-$ETCD_SHA256_PPC64}
elif is_arch "s390x"; then
    # An etcd3 binary for s390x is not available on github like it is
    # for other arches. Only continue if a custom download URL was
    # provided.
    if [[ -n "${ETCD_DOWNLOAD_URL}" ]]; then
        ETCD_ARCH="s390x"
        ETCD_SHA256=${ETCD_SHA256:-$ETCD_SHA256_S390X}
    else
        exit_distro_not_supported "etcd3. No custom ETCD_DOWNLOAD_URL provided."
    fi
elif is_arch "mips64"; then
    # An etcd3 binary for mips64 is not available on github like it is
    # for other arches. Only continue if a custom download URL was
    # provided.
    if [[ -n "${ETCD_DOWNLOAD_URL}" ]]; then
        ETCD_ARCH="mips64"
        ETCD_SHA256=${ETCD_SHA256:-$ETCD_SHA256_MIPS64}
    else
        exit_distro_not_supported "etcd3. No custom ETCD_DOWNLOAD_URL provided."
    fi
else
    exit_distro_not_supported "invalid hardware type - $ETCD_ARCH"
fi
```

脚本的下载辅助函数会判断文件是否已经存在，若存在则不下载，因此拷贝 `etcd-v3.3.12-linux-mips64.tar.gz` 到 `files` 目录即可，同时在 `local.conf` 中添加 `ETCD_DOWNLOAD_URL=https://fake-url`。

编译好的 ETCD 无法直接运行，因为官方没在 MIPS 上测试过，所以需要添加环境变量 `ETCD_UNSUPPORTED_ARCH=mips64le`。参考脚本 aarch64 体系结构部分的代码，直接把该环境变量添加到 `.service` 文件中。

```bash
# start_etcd3() - Starts to run the etcd process
function start_etcd3 {
    ...
    local unitfile="$SYSTEMD_DIR/$ETCD_SYSTEMD_SERVICE"
    write_user_unit_file $ETCD_SYSTEMD_SERVICE "$cmd" "" "root"

    iniset -sudo $unitfile "Unit" "After" "network.target"
    iniset -sudo $unitfile "Service" "Type" "notify"
    iniset -sudo $unitfile "Service" "Restart" "on-failure"
    iniset -sudo $unitfile "Service" "LimitNOFILE" "65536"
    if is_arch "aarch64"; then
        iniset -sudo $unitfile "Service" "Environment" "ETCD_UNSUPPORTED_ARCH=arm64"
    fi
    if is_arch "mips64"; then
        iniset -sudo $unitfile "Service" "Environment" "ETCD_UNSUPPORTED_ARCH=mips64le"
    fi

    $SYSTEMCTL daemon-reload
    $SYSTEMCTL enable $ETCD_SYSTEMD_SERVICE
    $SYSTEMCTL start $ETCD_SYSTEMD_SERVICE
}
```

### 修复已知问题

已知 lemote Fedora 28 系统上存在以下问题：

- 缺少 `dstat` 软件包，且该包并不会在脚本运行中自动安装，使用 `dnf` 安装即可。
- 系统自带的 `virtualenv` 版本过低，使用 `pip` 升级。
- 缺少 `uwsgi` 软件包，`dnf` 源中不存在该软件包，使用 `pip` 安装。
- 系统自带的 `wrapt` 会阻碍脚本自动升级该软件包，提前把它强制升级。

参考 `fixup_fedora` 函数，添加 `fixup_lemote` 函数，并在 `fixup_fedora` 函数中调用它。该部分代码位于 `tools/fixup_stuff.sh`。

```bash
function fixup_lemote {
    if ! is_lemote; then
        return
    fi

    # Install missing tools
    install_package dstat
    pip_install -U virtualenv
    pip_install -U uwsgi
    pip_install -UI wrapt
}

function fixup_fedora {
    ...
    fixup_lemote
}
```

### 创建虚拟环境

系统下的 Python3 不认识 `python3 -m venv` 命令，将其改为 `python3 -m virtualenv`。该部分代码位于 `stackrc`。

```bash
# Create a virtualenv with this
# Use the built-in venv to avoid more dependencies
if ! is_lemote; then
    export VIRTUALENV_CMD="python3 -m venv"
else # lemote
    export VIRTUALENV_CMD="python3 -m virtualenv"
fi
```

### 设置服务自启

系统重启后，DevStack 无法正常运行，检查日志和服务信息发现 `httpd` 和 `memcached` 服务没有设置自启。

检查脚本发现没有设置服务自启的封装函数（可能是我没找仔细？），参考 `start_service` 函数定义 `_enable_service` 函数 （注意已经存在 `enable_service` 函数了）。该部分代码位于 `functions-common`。

```bash
# Service wrapper to enable services
# Note that we already have enable_service above
# _enable_service service-name
function _enable_service {
    if [ -x /bin/systemctl ]; then
        sudo /bin/systemctl enable $1
    else
        sudo service $1 enable
    fi
}
```

之后，在对应的服务处加上自启。对于 `httpd`，脚本对 apache 服务又进行了一层封装，添加相应内容。代码位于 `lib/apache`。

```bash
# lib/apache exports the following functions:
# ...
# - enable_apache_server

# install_apache_wsgi() - Install Apache server and wsgi module
function install_apache_wsgi {
    ...
    enable_apache_server
}

# enable_apache_server() - Enabling apache server
function enable_apache_server {
    _enable_service $APACHE_NAME
}
```

对于 `memcached`，在安装处添加自启即可。代码位于 `lib/keystone`。

```bash
# start_keystone() - Start running processes
function start_keystone {
    ...
    _enable_service memcached
}
```

### 虚拟化

对于 lemote Fedora 28 系统，必须确保 `libvirt-python` 和 `python3-libvirt` 软件包的版本一致，且高于 6.6.0，索性直接在代码中写死。该部分代码位于 `lib/nova_plugins/functions-libvirt`。

```bash
# Installs required distro-specific libvirt packages.
function install_libvirt {

    if is_ubuntu; then
        install_package qemu-system libvirt-clients libvirt-daemon-system libvirt-dev
        if is_arch "aarch64"; then
            install_package qemu-efi
        fi
        # uninstall in case the libvirt version changed
        pip_uninstall libvirt-python
        pip_install_gr libvirt-python
        #pip_install_gr <there-si-no-guestfs-in-pypi>
    elif is_fedora || is_suse; then

        # Optionally enable the virt-preview repo when on Fedora
        if [[ $DISTRO =~ f[0-9][0-9] ]] && [[ ${ENABLE_FEDORA_VIRT_PREVIEW_REPO} == "True" ]]; then
            # https://copr.fedorainfracloud.org/coprs/g/virtmaint-sig/virt-preview/
            sudo dnf copr enable -y @virtmaint-sig/virt-preview
        fi

        # Note that in CentOS/RHEL this needs to come from the RDO
        # repositories (qemu-kvm-ev ... which provides this package)
        # as the base system version is too old.  We should have
        # pre-installed these
        install_package qemu-kvm

        install_package libvirt libvirt-devel
        if is_arch "aarch64"; then
            install_package edk2.git-aarch64
        fi

        if ! is_lemote; then
            pip_uninstall libvirt-python
            pip_install_gr libvirt-python
        else # lemote
            install_package libvirt-6.6.0
            install_package python3-libvirt-6.6.0
        fi
    fi

    if [[ $DEBUG_LIBVIRT_COREDUMPS == True ]]; then
        _enable_coredump
    fi
}
```

同时，系统只支持 `custom` 模式，模拟 `Loongson-3A4000`（见[修改 CPU 模式](#修改-cpu-模式)部分）。默认的脚本只提供 `none`、`host-model` 和 `host-passthrough` 模式，参考 aarch64 体系结构，添加相关代码。该部分代码位于 `lib/nova_plugins/hypervisor-libvirt`。

```bash
# configure_nova_hypervisor - Set config files, create data dirs, etc
function configure_nova_hypervisor {
    ...
    # arm64-specific configuration
    if is_arch "aarch64"; then
        iniset $NOVA_CONF libvirt cpu_mode "host-passthrough"
    fi

    # mips64-specific configuration
    if is_arch "mips64"; then
        iniset $NOVA_CONF libvirt cpu_mode "custom"
        iniset $NOVA_CONF libvirt cpu_models "Loongson-3A4000"
    fi
    ...
}
```

## 移植 Nova

由于龙芯平台的 libvirt 和 QEMU/KVM 的实现还不完全，nova-compute 不经过修改无法在龙芯平台上正常运行。修改过的 Nova 代码见 <https://github.com/FreeFlyingSheep/Nova>。

### 修改 CPU 模式

在未修改 CPU 模式时，运行 DevStack，尝试创建虚拟机时报错，查看日志显示 libvirt 内部错误，未知 CPU 类型 `qemu64`。

查看 nova-compute 配置文件发现 DevStack 默认 CPU 模式设置为 `none`，尝试改为 `host-model` 和 `host-passthrough`，依然报 libvirt 内部错误，但错误内容变为不支持这两种模式。

查看 libvirt 源码，定位报错的位置，发现龙芯平台上的 libvirt 实现并不完全，去把这些接口都实现了显然不现实。考虑到龙芯平台通过修改过的 virt-manager 是能正常创建虚拟机的，就想到比较龙芯平台的 virt-manager 创建虚拟机的参数和 nova-compute 默认创建的参数（主要比较生成的给 libvirt 使用的 xml 文件）。

对比发现，virt-manager 生成的 xml 文件中，使用的是 `custom` 模式，模拟 `Loongson-3A4000`。修改 nova-compute 的配置文件，设置这两个参数，错误变为了没有使用 PCIe。

### 修改 PCIe 配置

查看 Nova 源码，直接强制检查 PCIe 的函数对于 mips64el 架构返回 `True`。该部分代码位于 `virt/libvirt/driver.py`。

```python
def _guest_needs_pcie(self, guest, caps):
    """Check for prerequisites for adding PCIe root port
    controllers
    """
    ...
    if caps.host.cpu.arch == fields.Architecture.MIPS64EL:
        return True
    if not CONF.libvirt.num_pcie_ports:
        return False
    if (caps.host.cpu.arch == fields.Architecture.AARCH64 and
            guest.os_mach_type.startswith('virt')):
        return True
    if (caps.host.cpu.arch == fields.Architecture.X86_64 and
            guest.os_mach_type is not None and
            'q35' in guest.os_mach_type):
        return True
    return False
```

重启 Nova 服务，发现虚拟机创建成功，但无法运行，错误变为了没法加载 BIOS。

### 添加其他参数

继续比较 xml 文件，发现 virt-manager 生成的 xml 会指定 BIOS 文件和其他相关参数，查看 Nova 源码，将这部分内容写到对应位置。代码位于 `virt/libvirt/driver.py`。

```python
def _configure_guest_by_virt_type(self, guest, virt_type, caps, instance,
                                  image_meta, flavor, root_device_name,
                                  sev_enabled):
    if virt_type == "xen":
        ...
    elif virt_type in ("kvm", "qemu"):
        if caps.host.cpu.arch in (fields.Architecture.I686,
                                    fields.Architecture.X86_64):
            guest.sysinfo = self._get_guest_config_sysinfo(instance)
            guest.os_smbios = vconfig.LibvirtConfigGuestSMBIOS()
        hw_firmware_type = image_meta.properties.get('hw_firmware_type')
        ...
        if image_meta.properties.get('hw_boot_menu') is None:
            guest.os_bootmenu = strutils.bool_from_string(
                flavor.extra_specs.get('hw:boot_menu', 'no'))
        else:
            guest.os_bootmenu = image_meta.properties.hw_boot_menu
        if caps.host.cpu.arch == fields.Architecture.MIPS64EL:
            guest.os_loader = "/usr/share/qemu/bios_loongson3.bin"
            guest.os_loader_type = "rom"
            guest.os_mach_type = "loongson3-virt"

    elif virt_type == "lxc":
        ...
```

重启 Nova 服务，发现虚拟机成功创建，并运行，但控制台界面卡在了加载内核。

### 修改显示模式（后来又改回去了）

查看日志，发现并没有任何报错。思考了一会，推测是内核其实已经成功启动，但没正常显示。

继续对比 xml 文件，发现 nova-compute 默认使用的显卡设备模型为 `cirrus`，而 virt-manager 使用的显卡设备模型为 `qxl`。在 virt-manager 的选项中，没有找到 `cirrus` 显卡设备模型，怀疑不支持。

于是修改 nova-compute 相关代码，使它在 mips64 架构下默认使用 `qxl`。该部分代码位于 `virt/libvirt/driver.py`。

```python
def _add_video_driver(self, guest, image_meta, flavor):
    ...
    if guest.os_type == fields.VMMode.XEN:
        video.type = 'xen'
    elif CONF.libvirt.virt_type == 'parallels':
        video.type = 'vga'
    elif guestarch in (fields.Architecture.PPC,
                        fields.Architecture.PPC64,
                        fields.Architecture.PPC64LE):
        # NOTE(ldbragst): PowerKVM doesn't support 'cirrus' be default
        # so use 'vga' instead when running on Power hardware.
        video.type = 'vga'
    elif guestarch == fields.Architecture.AARCH64:
        # NOTE(kevinz): Only virtio device type is supported by AARCH64
        # so use 'virtio' instead when running on AArch64 hardware.
        video.type = 'virtio'
    elif (CONF.spice.enabled) or (guestarch == fields.Architecture.MIPS64):
        video.type = 'qxl'
    if image_meta.properties.get('hw_video_model'):
        video.type = image_meta.properties.hw_video_model
        if not self._video_model_supported(video.type):
            raise exception.InvalidVideoMode(model=video.type)
    ...
```

重启 Nova 服务，这次控制台界面显示出了开机动画，但发现键盘没有反应。

### 添加键盘设备

鼠标能动但键盘没有反应，推测使没添加键盘，查看 xml 发现果然如此。

在 Nova 中添加键盘。该部分代码位于 `virt/libvirt/driver.py`。

```bash
def _get_guest_config(self, instance, network_info, image_meta,
                      disk_info, rescue=None, block_device_info=None,
                      context=None, mdevs=None, accel_info=None):
    """Get config data for parameters.

    :param rescue: optional dictionary that should contain the key
        'ramdisk_id' if a ramdisk is needed for the rescue image and
        'kernel_id' if a kernel is needed for the rescue image.

    :param mdevs: optional list of mediated devices to assign to the guest.
    :param accel_info: optional list of accelerator requests (ARQs)
    """
    ...
    self._guest_add_spice_channel(guest)

    if self._guest_add_video_device(guest):
        self._add_video_driver(guest, image_meta, flavor)

        # We want video == we want graphical console. Some architectures
        # do not have input devices attached in default configuration.
        # Let then add USB Host controller and USB keyboard.
        # x86(-64) and ppc64 have usb host controller and keyboard
        # s390x does not support USB
        if caps.host.cpu.arch in (fields.Architecture.AARCH64,
                                    fields.Architecture.MIPS64EL):
            self._guest_add_usb_host_keyboard(guest)

    # Some features are only supported 'qemu' and 'kvm' hypervisor
    if virt_type in ('qemu', 'kvm'):
        self._set_qemu_guest_agent(guest, flavor, instance, image_meta)
        self._add_rng_device(guest, flavor, image_meta)
        self._add_vtpm_device(guest, flavor, instance, image_meta)
    ...
```

重启 Nova 服务，键盘能正常使用了，但鼠标能移动却无法点击。

### 改用 spice

听取同事建议后，索性改用 spice 协议，而不是 vnc 协议。这样可以移除 Nova 中显示模式部分的修改，鼠标也能正常使用了。

对于 DevStack，只需要修改配置文件 `local.conf` 即可，添加两行内容：

```text
enable_service n-spice n-sproxy
disable_service n-novnc n-xvnc
```

## 其他问题

创建虚拟机的过程中，一旦涉及 Cinder 服务，就会报错，因此暂时使用了 Nova 自带的创建卷功能。

成功把 Nova 跑通后，开始解决 Cinder 无法映射块设备的问题。查看日志报错信息，发现是创建 LVM 卷的时候出错了。

根据网上资料手动创建 LVM 卷，复现了报错信息，只要是使用 `thin` 模式，创建卷就会报错，这应该是系统中相关的软件包存在问题，暂时将 cinder 的 LVM 模式更换为 `default`，绕过了这个问题，成功创建了卷。

还有一些问题，但暂时没想到解决办法。比如内核没有输出日志信息，检查 xml 确定已经重定向到了 `console.log`，内核启动也添加了相关参数。还有从 ISO 文件创建虚拟机时会报找不到设备等问题（由于系统的 QEMU/KVM 只支持龙芯的镜像，所以必须使用提前准备好的镜像）。

对于 DevStack，最终的 `local.conf` 配置如下：

```text
[[local|localrc]]
FORCE=yes

ADMIN_PASSWORD=123456
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

GIT_BASE=https://github.com
NOVA_REPO=https://github.com/FreeFlyingSheep/nova.git

ETCD_DOWNLOAD_URL=https://fake-url

DOWNLOAD_DEFAULT_IMAGES=False
IMAGE_URLS=https://fake-url/fedora28.qcow2

CINDER_LVM_TYPE=default

enable_service n-spice n-sproxy
disable_service n-novnc n-xvnc
```

至此，OpenStack 的移植初步完成了，到了勉强能用的地步。因为暂时没有后续的需求，所以先到此为止了。
