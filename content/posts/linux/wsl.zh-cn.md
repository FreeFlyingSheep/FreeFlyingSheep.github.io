---
title: "安装 WSL"
date: 2020-09-21
description: ""
menu:
  sidebar:
    name: "安装 WSL"
    identifier: linux-wsl
    parent: linux
    weight: 100
tags: ["WSL"]
categories: ["Linux"]
---

根据微软官方文档 [Windows Subsystem for Linux Installation Guide for Windows 10](https://docs.microsoft.com/en-us/windows/wsl/install-win10) 整理。

<!--more-->

## WSL 1 和 WSL 2 的选择

两者的比较可以参考微软官方文档[比较 WSL 1 和 WSL 2](https://docs.microsoft.com/zh-cn/windows/wsl/compare-versions)。

个人推荐优先选择 WSL 2，若发现 WSL 2 和其他软件冲突，再回退到 WSL 1。

## 启用 WSL 1

以管理员身份打开 PowerShell 并运行：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

重启计算机即可。

## 启用 WSL 2

在[启用 WSL 1](#启用-wsl-1)的基础上，不要重启，而是以管理员身份打开 PowerShell 并运行：

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

现在重启计算机。

## 安装升级软件包

下载 [WSL2 Linux kernel update package for x64 machines](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi) 并安装。

## 将 WSL 2 设置为默认版本

以管理员的身份打开 PowerShell 并运行：

```powershell
wsl --set-default-version 2
```

## 从 WSL 2 回到 WSL 1

当 WSL 2 和其他软件冲突，需要关闭相关功能时，以管理员的身份打开 PowerShell 并运行：

```powershell
wsl --set-default-version 1
dism.exe /online /disable-feature /featurename:VirtualMachinePlatform
```

重启计算机。

## 关闭 WSL 1

若开启了 WSL 2，先[从 WSL 2 回到 WSL 1](#从-wsl-2-回到-wsl-1)，跳过重启。

以管理员的身份打开 PowerShell 并运行：

```powershell
dism.exe /online /disable-feature /featurename:Microsoft-Windows-Subsystem-Linux
```

现在重启计算机。

## 安装 Linux 发行版

在应用商店选择想要安装的发行版即可。个人推荐在 WSL 1 下使用 Debian，在 WSL 2 下使用 Ubuntu 或者 Debian。

## 指定发行版使用 WSL1 或 WSL2

查看所有安装的发行版，以管理员的身份打开 PowerShell 并运行：

```powershell
wsl --list -v
```

举个例子，若需要修改发行版 Debian 使用 WSL 1，运行以下指令：

```powershell
wsl --set-version Debian 1
```

## 安装 Windows Terminal

在应用商店选择安装 Windows Terminal。

若想把 WSL 设为默认启动的终端，将 `defaultProfile` 设置为 WSL 的 `guid`，默认为 PowerShell。

如果要指定终端启动时的大小，在 `defaultProfile` 之后加入如下配置（配置文件提供的注释已经很友好了）：

```json
// You can add more global application settings here.
// To learn more about global settings, visit https://aka.ms/terminal-global-settings
"initialCols": 80,
"initialRows": 24,
```

在 `profiles` 的 `defaults` 中加入通用的配置，如：

```json
// Put settings here that you want to apply to all profiles.
"fontFace": "Jetbrains Mono",
"fontSize": 14,
"colorScheme": "idleToes"
```

若使用自定义主题（如上面的 `idleToes`），需要在 `schemes` 中加入与 `colorScheme` 对应的颜色主题配置：

```json
// Add custom color schemes to this array.
// To learn more about color schemes, visit https://aka.ms/terminal-color-schemes
"schemes": [
    {
        "name": "idleToes",
        "black": "#323232",
        "red": "#d25252",
        "green": "#7fe173",
        "yellow": "#ffc66d",
        "blue": "#4099ff",
        "purple": "#f680ff",
        "cyan": "#bed6ff",
        "white": "#eeeeec",
        "brightBlack": "#535353",
        "brightRed": "#f07070",
        "brightGreen": "#9dff91",
        "brightYellow": "#ffe48b",
        "brightBlue": "#5eb7f7",
        "brightPurple": "#ff9dff",
        "brightCyan": "#dcf4ff",
        "brightWhite": "#ffffff",
        "background": "#323232",
        "foreground": "#ffffff"
    }
]
```

更多颜色主题可以参考 <https://windowsterminalthemes.dev/>。

如果希望打开终端时路径是家目录，下面介绍两种常见的方式。

第一种方法是很多博客介绍的，将 WSL 对应的 `source` 改为 `commandline`，并添加 `startingDirectory`，以 Ubuntu 20.04 为例，本人的家目录是 `/home/chris`，修改配置如下：

```json
// "source": "Windows.Terminal.Wsl"
"commandline": "ubuntu2004",
"startingDirectory": "/home/chris"
```

第二种方法是官方文档里介绍的，只需要添加 `startingDirectory` 选项，以 Debian 为例，修改配置如下：

```json
"source": "Windows.Terminal.Wsl",
"startingDirectory": "//wsl$/debian"
```

## WSL2 和部分 VPN 冲突

使用 WSL2 时出现“参考的对象类型不支持尝试的操作”的提示，可能是因为开启了 VPN，解决方案如下：

>Thanks for the info.
>
>We have reproduced this issue.
>Apparently, wsl.exe displays this error if Winsock LSP DLL gets loaded into its process.
>
>The easiest solution is to use WSCSetApplicationCategory WinAPI call for wsl.exe to prevent >this.
>Under the hood the call creates an entry for wsl.exe at >HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\WinSock2\Parameters\AppId_Catalog
>This tells Windows not to load LSP DLLs into wsl.exe process.
>
>We have a tool that can make this call:
>www.proxifier.com/tmp/Test20200228/NoLsp.exe
>
>Please just run as admin with the full path to wsl.exe as the parameter:
>NoLsp.exe c:\windows\system32\wsl.exe
>
>This has fixed the problem in my case.
>
>Please let me know how it works for you.

上述内容来源于 <https://github.com/microsoft/WSL/issues/4177#issuecomment-597736482>。
