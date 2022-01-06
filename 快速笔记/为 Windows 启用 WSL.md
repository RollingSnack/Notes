---
Title: 为 Windows 启用 WSL
Tags: [类型/教程, 操作系统/Windows, 操作系统/WSL, 操作系统/Linux]
---

# 为 Windows 启用 WSL

![[WSL]]

## 环境

本文主要参考 [官方文档](https://docs.microsoft.com/en-us/windows/wsl/install-win10) 的手动安装部分，且目标为 WSL2。

目前 WSL 只可工作在 [[Windows]] 10 上，对于 x64 系统的要求是 Version 1903 以上且 Build 18362 以上。

## 安装

### 启用 WSL

WSL 是一个可选功能，已经内置于 Windows 10 中。
需要做的只是启用这一功能。

以**管理员身份**运行 [[PowerShell]]，然后执行命令：

```PowerShell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

### 安装 WSL2

这里目标使用的是 WSL2，与 WSL 相比，它是一个虚拟化解决方案，因此还需要启用 [[虚拟机]] 功能。

```PowerShell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

之后从微软官方下载 [Linux 内核安装包 x64 版本](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)（点击即下载），然后安装。

安装完后将 WSL2 设置为默认版本替代 WSL。

```PowerShell
wsl --set-default-version 2
```

### 下载 Linux 发行版

[[Linux]] 系统还需要从 [Microsoft Store](https://aka.ms/wslstore)（点击即启动商店 APP）中额外下载安装。

这里选用的 [[发行版]] 是 [[Ubuntu]]。
直接使用不带后缀的 Ubuntu 会默认安装最新版本的 Ubuntu。
初次使用时，会弹出一个控制台界面。
需要为新系统创建一个新用户并设置密码。

## 运行

由于一个发行版就是一个 `exe` 文件，调用它的方式和 Windows 下的众多软件无异。

简单来说如下：
1. GUI 模式。找到安装好了的发行版，打开运行即可。这种方式下会启动系统默认的 [[shell]]，如 [[cmd]]。
2. 命令行模式。输入对应发行版的名字，如 `ubuntu` 或 `ubuntu1804` 等。
3. 命令行模式。Windows 还提供了管理工具 `wsl`，可以通过它启动所需的发行版；不加任何参数时会指向默认的发行版。如果只管理发行版，还可以使用 `wslconfig`，它是前者的子集。

## 讨论

WSL2 实现靠的是虚拟机，它和 Linux 实机乃至 WSL 都还有不少的区别，不可能做到赢者通吃。
许多文章对此也有过介绍，观察下来，主要在接口、性能、网络等方面上有部分问题。
不过好消息是，WSL2 还在不断地迭代更新，一些短板或许会在未来版本中加强。

后续文章将根据个人的使用情况，记录部分遭遇的问题及可能的解决方案。