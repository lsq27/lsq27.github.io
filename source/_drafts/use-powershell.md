---
title: 增强 Windows 开发体验
date: 2023-04-09 13:12:36
tags: [Windows, package manager, Scoop, PowerShell]
categories: [工具]
description: 绿色便携软件爱好者的福音 - Scoop，在 Windows 下也可以享受 Linux 的包管理工具的快乐
---

Windows 的终端相比于 Unix-Like 来说，一直处于弱势。微软注意到了这个问题，在尝试增强 Windows 下的开发体验。最近正好重装了笔记本的操作系统

一般来说，PowerShell 中用户创建的变量、函数等只能在当前的窗口会话生效。但配置文件非常特殊，PowerShell 每次启动前都会加载该文件。如果将某些命令写入这个文件，修改就能全局生效。新建或修改配置文件最简单的方法是输入 notepad $profile，在弹出的编辑窗口添加命令即可。

## 包管理

个人喜欢的包管理工具是 [Scoop](https://scoop.sh/)，主打便携功能当然 Chocolate 和 Winget 也是不错的选择。

## 命令行

### Nerd Fonts

安装 一款 [Nerd Fonts](https://github.com/ryanoasis/nerd-fonts)，推荐 `Fira Code Nerd Font`，`Fira Code` 支持连字 `ligature`，`Nerd Fonts` 在原有字体的基础上增加了图标字符。

### PowerShell

安装最新版 [PowerShell 7.3.3](https://www.microsoft.com/store/productId/9MZ1SNWT0N5D)

### Windows Terminal

安装最新版 [Windows Terminal 1.15](https://www.microsoft.com/store/productId/9N0DX20HK701)

- 将 `PowerShell` 设置为默认配置文件
  ![PowerShell](images/1.png)
- 将 `Fira Code NF` 设置为默认字体
  ![PowerShell](images/2.png)

## PowerShell

[PSReadLine](https://github.com/PowerShell/PSReadLine)

```powershell
Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
scoop install scoop-completion
```

## 利用别名缩短命令

获取别名

```powershell
> Get-Alias name
```

获取别名

```powershell
> Get-Alias name
```

设置别名

```powershell
> Set-Alias alias command
```

移除别名

```powershell
> Remove-Alias name
```
