---
title: 使用MSYS2
date: 2023-04-08 20:39:43
tags: [Windows, package manager, Scoop, PowerShell]
categories: [工具]
description: MSYS2 是适用于 Windows 的软件分发和构建平台，用于管理软件，提供原生 Windows 构建环境
---

## MSYS2 是什么

来自官网的介绍

> MSYS2 是工具和库的集合，为您提供易于使用的环境，用于构建、安装和运行原生 Windows 软件。

它由 [mintty](https://mintty.github.io) terminal，bash shell，和一些基于 [Cygwin](https://cygwin.com) 的工具组成。
MSYS2 可以为 Windows 提供最新的 GCC、mingw-w64、 CPython，CMake，Meson，OpenSSL，FFmpeg，Rust，Ruby 等软件最新的原生构建。

他的包管理工具是来自 Arch Linux 的 [Pacman](https://wiki.archlinux.org/index.php/pacman)，现有约 2800 个软件。

## MSYS2 为什么要用 MSYS2

我最开始用 MSYS2 的目的很简单，需要在 Windows 下编译 C++，不喜欢 MSVC 的臃肿于是选择了 [MinGW-w64](https://www.mingw-w64.org)，MinGW-w64 的下载页中是 Windows 平台且提供的软件较多的两个安装包分别是 Cygwin 和 MSYS2。比较了一番他们的区别之后，我选择了提供原生 Windows 构建工具的 MSYS2。

## 快速开始

### 安装 MSYS2

使用 Scoop 、 Winget 或[官网](https://www.msys2.org)安装包

```powershell
> scoop install msys2
> winget install MSYS2.MSYS2
```

### 更新 MSYS2

启动 UCRT64，更新包，执行 `pacman -Syuu` 多次直到提示无更新可用。

### 搜索包

搜索可用软件包

```bash
$ pacman -Ss <name_pattern>
```

### 安装包

安装软件包或一组软件包

```bash
$ pacman -S <package_names|package_groups>
```

###

列出显示安装过的包

```
pacman -Qe
```

### 卸载包

卸载软件包或一组软件包

```bash
$ pacman -R <package_names|package_groups>
```

## 开发配置

### 安装工具链

安装 MSYS2 最大的目的就是 GCC 工具链，其中有两个软件包组需要安装

- [base-devel](https://packages.msys2.org/package/base-devel?repo=msys&variant=x86_64) 包含 make，curl，grep 等开发常用的基础包
- [mingw-w64-ucrt-x86_64-toolchain](https://packages.msys2.org/groups/mingw-w64-ucrt-x86_64-toolchain) 包含 GCC 工具链

和 Linux 相似，`devel` 结尾的包，包含使用该程序进行开发的所有的必需文件，例如 libcurl-devel 中包含 Libcurl 的头文件和库文件

`msys repo` 下的包会被安装到 `/usr` 目录下，`ucrt64 repo` 下的包会被安装到 `/ucrt64`

```bash
pacman -S --needed base-devel mingw-w64-ucrt-x86_64-toolchain
```

### 更换 shell

MSYS2 默认为 Bash，个人更喜欢 Fish。安装配置方式与 Linux 下相同

```bash
$ pacman -S fish
```

```bash
path/to/msys2_shell.cmd -defterm -here -no-start -ucrt64 -shell fish
```

> 注意：更换 shell 后如果还需运行 Bash 脚本，请在 Bash 脚本第一行添加
>
> ```bash
> #!/usr/bin/env bash
> ```

```bash
pacman -S mingw-w64-ucrt-x86_64-oh-my-posh
```

```bash
nano ~/.config/fish/config.fish
```

```bash
oh-my-posh init fish --config /ucrt64/share/oh-my-posh/themes/1_shell.omp.json | source
```

## 进阶

### 交叉编译

### 持续集成

MSYS2 带有不同的环境/子系统，您拥有的第一件事 决定使用哪一个。环境之间的差异是 主要是环境变量、默认编译器/链接器、架构、 使用的系统库等如果您不确定，请选择 UCRT64。

MSYS 环境包含基于 unix/cygwin 的工具，存在于其下，并且特别之处在于它始终处于活动状态。所有其他环境 从 MSYS 环境继承并在其上添加各种内容。/usr

例如，在 UCRT64 环境中，变量以 因此您可以获得所有基于 ucrt64 的工具以及所有 msys 工具。`$PATH=/ucrt64/bin:/usr/bin`
