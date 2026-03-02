---
title: "安装 Arch Linux"
date: 2020-10-20
description: ""
menu:
  sidebar:
    name: "安装 Arch Linux"
    identifier: linux-arch
    parent: linux
    weight: 100
tags: ["Arch Linux"]
categories: ["Linux"]
---

根据 [Arch Linux Wiki](https://wiki.archlinux.org/) 整理 Arch Linux 的安装。

<!--more-->

## 安装前准备

参考 [Installation guide](https://wiki.archlinux.org/index.php/Installation_guide)。

### 准备安装介质

从[下载页面](https://www.archlinux.org/download/)下载镜像，并从安装介质启动。

目前我使用的镜像是 `archlinux-2020.10.01-x86_64.iso`。

### 设置键盘布局

我采用默认布局（美式键盘）。

### 验证引导模式

```bash
ls /sys/firmware/efi/efivars
```

如果该命令没有报错，说明系统是 UEFI 模式引导的，反之是传统方式引导的。

我的系统是 UEFI 模式引导的。

### 连接到网络

我用的是有线连接，使用 `ip link` 命令可以查看到对应的网卡在列表中，并且已经启用。

测试 `ping archlinux.org` 成功。

### 更新系统时钟

```bash
timedatectl set-ntp true
```

### 磁盘分区

使用 `fdisk -l` 查看现在的磁盘分区。

我需要重新分区，使用 `parted` 命令，参考 [Parted](https://wiki.archlinux.org/index.php/Parted) 完成分区：

1. `parted /dev/sda` 进入 `parted`。
2. `mklabel` 设置 `gpt` 分区类型。
3. `mkpart "EFI system partition" fat32 1MiB 301MiB` 设置 EFI 分区。
4. `set 1 esp on` 设置 EFI 分区标志。
5. `mkpart "root partition" ext4 301MiB 102701MiB` 设置根分区。
6. `mkpart "swap partition" linux-swap 102701MiB 110893MiB` 设置交换分区。
7. `mkpart "home partition" ext4 110893Mib 100%` 设置 `/home` 分区。
8. `print` 确认分区无误，若发现问题，用 `rm <分区号>` 删除，再重新用 `mkpart` 分区。
9. `quit` 退出 `parted`

### 格式化分区

再次使用 `fdisk -l` 查看现在的磁盘分区。

执行以下命令格式化分区：

```bash
mkfs.vfat /dev/sda1
mkfs.ext4 /dev/sda2
mkswap /dev/sda3
mkfs.ext4 /dev/sda4
```

### 挂载文件系统

挂载根文件系统到 `/mnt`：

```bash
mount /dev/sda2 /mnt
```

挂载 EFI 分区到 `/mnt/boot/efi`：

```bash
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```

挂载 `/home` 分区到 `/mnt/home`：

```bash
mkdir /mnt/home
mount /dev/sda4 /mnt/home
```

启用交换分区：

```bash
swapon /dev/sda3
```

## 安装系统

参考 [Installation guide](https://wiki.archlinux.org/index.php/Installation_guide)。

### 选择镜像服务器

执行 `vim /etc/pacman.d/mirrorlist` 修改镜像列表。

把中国的站点放到最前面，发现清华站点竟然就在第一个的位置，直接退出即可。

### 安装必要的软件包

这里只安装了一些必要的软件包，其余软件包可以稍后安装。

```bash
pacstrap /mnt base linux linux-firmware networkmanager vi vim man-db man-pages texinfo
```

### 配置系统

### 配置 `fstab` 文件

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

`cat /mnt/etc/fstab` 确认文件内容无误。

### 进入 `Chroot` 环境

```bash
arch-chroot /mnt
```

### 配置时区

我需要设置上海时区：

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```

### 本地化配置

执行 `vim /etc/locale.gen` 把需要的 locales 前的注释去掉。

我去掉了 `en_US.UTF-8 UTF-8` 和 `zh_CN.UTF-8 UTF-8` 前面的注释。

执行 `locale-gen` 生成 locales。

执行 `vim /etc/locale.conf` 创建并编辑文件内容为 `LANG=en_US.UTF-8`。

注意，默认使用英文，以后我也只在图形界面使用中文，来避免乱码问题。

### 网络配置

执行 `vim /etc/hostname` 创建并编辑文件内容为主机名，我使用的是 `loongson`。

执行 `vim /etc/hosts` 添加相应内容，注意与主机名对应：

```text
127.0.0.1        localhost
::1              localhost
127.0.1.1        loongson.localdomain   loongson
```

### Initramfs

这步可以直接跳过。

### 设置 `root` 密码

```bash
passwd
```

### 安装引导程序

我选择使用 GRUB，参考 [GRUB](https://wiki.archlinux.org/index.php/GRUB)。

由于我的计算机使用的是 Intel CPU，所以要安装相应的固件 `intel-ucode`。

```bash
pacman -S intel-ucode grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

## 重启系统

确认上述所有步骤都完成后可以重启。

```bash
exit
reboot
```

若重启后无法进入系统，除非你能排查问题并修复，否则建议直接重来。

## 安装开发环境

### 启用网络

启用网络并配置：

```bash
systemctl start NetworkManager
```

设置开机启动：

```bash
systemctl enable NetworkManager
```

### 安装常用软件包

```bash
pacman -S base-devel zsh gdb git
```

## 配置用户

### 添加用户

参考 [Users and groups](https://wiki.archlinux.org/index.php/Users_and_groups)。

创建新用户 `loongson`，配置其登录 shell 为 bash，以后再使用 zsh：

```bash
useradd -m -s /bin/bash loongson
```

为 `loongson` 配置密码：

```bash
passwd loongson
```

### 设置管理员权限

将 `loongson` 添加到 `wheel` 组：

```bash
usermod -a -G wheel loongson
```

编辑相应的配置文件，删除 `# %wheel ALL=(ALL) ALL` 一行的注释：

```bash
EDITOR=vim visudo
```

## 安装图形界面

### 安装显示驱动

我用的是 Intel 集成显卡：

```bash
pacman -S xf86-video-intel mesa
```

### 安装显示服务器

我选择使用 `Xorg`，参考 [Xorg](https://wiki.archlinux.org/index.php/Xorg)。

```bash
pacman -S xorg
```

### 安装桌面环境

我选择使用 `KDE`，参考 [KDE](https://wiki.archlinux.org/index.php/KDE)。

```bash
pacman -S plasma kde-applications
```

设置开机启动：

```bash
systemctl enable sddm
```

重启系统进入图形界面：

```bash
reboot
```

## 其他操作

下面是可选的其他操作，仅针对 KDE 桌面环境。

### 缩放屏幕

我需要修改屏幕缩放到 125%，在 `System Settings -> Hardware -> Display and Monitor -> Display Configuration` 中设置。

设置完后先不急着重启，等后续步骤完成再一起重启。

### 安装中文字体

参考 [Localization (简体中文)/Simplified Chinese (简体中文)](https://wiki.archlinux.org/index.php/Localization_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)/Simplified_Chinese_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))。

打开 `Konsole`，我选择安装文泉驿中文字体：

```bash
pacman -S wqy-microhei wqy-bitmapfont wqy-zenhei
```

### 设置图形界面显示中文

修改 `System Settings -> Personalization -> Regional Settings -> Language`，添加简体中文到顶部（可能显示的是部分乱码，暂时不用管）。

现在重启计算机，进入桌面后应该是中文环境，且没有乱码，之后可以完成的其他个性化配置。

### 添加中国社区源

我选择使用清华源，在 `/etc/pacman.conf` 中添加以下内容：

```text
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

安装密钥：

```bash
sudo pacman -S archlinuxcn-keyring
```

如果出现密钥无法部署的问题，可以参考 <https://www.archlinuxcn.org/gnupg-2-1-and-the-pacman-keyring/>，进行如下操作：

```bash
pacman -Syu haveged
systemctl start haveged
systemctl enable haveged

rm -fr /etc/pacman.d/gnupg
pacman-key --init
pacman-key --populate archlinux
pacman-key --populate archlinuxcn
```

### 使用 AUR

我选择安装 yay：

```bash
sudo pacman -S yay
```

### 安装中文输入法

我选择安装搜狗输入法，该输入法在 AUR 中。

首先，安装 `fcitx`：

```bash
sudo pacman -S fcitx kcm-fcitx
```

其次，安装搜狗输入法：

```bash
yay -S fcitx-sogoupinyin
```

然后，新建 `~/.pam_environment` 并添加以下内容：

```text
GTK_IM_MODULE DEFAULT=fcitx
QT_IM_MODULE  DEFAULT=fcitx
XMODIFIERS    DEFAULT=\@im=fcitx
```

最后，注销后重新登录，确认 fcitx 正常启动，右键修改配置，之后重新启动 fcitx，搜狗输入法应该能正常使用。

### 美化终端

安装 oh-my-zsh，见 [使用 Zsh](/zh-cn/posts/tips/zsh)。

### 配置开机动画

参考 [Plymouth](https://wiki.archlinux.org/index.php/Plymouth)。

安装 Plymouth：

```bash
sudo pacman -S plymouth
```

修改 `/etc/mkinitcpio.conf`：

```text
MODULES=(i915 ...)
...
HOOKS=(base udev plymouth ...)
```

修改 `/etc/default/grub`：

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 rd.udev.log_priority=3 vt.global_cursor_default=0"
```

重新生成 GRUB 配置文件：

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

安装 Plymouth 主题：

```bash
yay -S plymouth-theme-arch-glow
sudo plymouth-set-default-theme -R arch-glow
```

设置平滑过渡：

```bash
sudo systemctl disable sddm
sudo systemctl enable sddm-plymouth
```

重启计算机查看开机动画。

### 配置 GRUB 主题

安装 [Vimix](https://github.com/vinceliuice/grub2-themes/) 主题。

重启查看效果。
