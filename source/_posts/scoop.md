---
title: Windows 下最好的便携式包管理工具 - Scoop
date: 2022-02-07 12:51:14
tags: [Windows, package manager, Scoop, PowerShell]
description: 绿色便携软件爱好者的福音 - Scoop，在 Windows 下也可以享受 Linux 的包管理工具的快乐
---

## Scoop 是什么

[Scoop](https://scoop.sh/)是一个 Windows 下的命令行安装器。其官网的自我介绍是

> Scoop 以阻力最小的方式从命令行安装您熟悉和喜爱的程序，它可以：
>
> - 消除权限弹出窗口
> - 隐藏安装程序的界面
> - 防止安装大量程序对 PATH 的污染
> - 避免安装和卸载程序的意外副作用
> - 查找并安装依赖项
> - 执行所有额外的程序设置步骤

## 为什么要用 Scoop

以前我要安装一个软件，我会

- 搜索引擎搜索 node
- 打开官网 (注意避开广告！)
- 寻找适合版本的下载链接，使用下载工具下载
- 打开安装包配置我需要的参数并安装

通过 Scoop 安装一个软件的，只需输入一个命令，Scoop 帮我们

- 下载安装包(压缩包)
- 安装(解压)
- 链接安装目录
- 持久化文件
- 执行脚本

Scoop 不只帮我们省略了找到安装包的过程，还免去了安装时的配置过程(Scoop 在安装脚本中写明了最佳的安装配置)，更妙的是全程都没用到管理员权限，自然也不必担心安装包对操作系统未知的修改(比如偷偷装个证书，设置自己为开机启动)。

和自行安装软件相比，我认为 Scoop 的优点有

- 软件数量多，官方桶中包含约 4000 个软件，第三方桶也有丰富应用
- 安装不依赖管理员权限，除了极少数特殊软件需要修改注册表或者安装到管理员目录
- 自动配置，安装时采用默认配置，Scoop 默认把路径设置为自己的目录，开机自启动等设置为否，简化安装步骤
- 自动安装依赖
- 自动添加命令行程序的链接到 `~/scoop/shims` ，然后再将该目录添加到环境变量，自动添加 GUI 程序的快捷方式到开始菜单
- 比 Linux 的包管理工具更便携的软件管理体验，即使重装系统也能快速恢复软件环境

## 快速开始

### 安装 Scoop

打开 PowerShell (版本号大于等于 5.1)，执行以下命令

```powershell
> Set-ExecutionPolicy RemoteSigned -Scope CurrentUser # 获取执行脚本权限
> irm get.scoop.sh | iex
> irm get.scoop.sh -Proxy 'http://<ip:port>' | iex # 使用代理连接 GitHub
```

Scoop 会被安装到 `C:\Users\<YOUR USERNAME>\scoop` 目录下，如果想自定义安装目录

```powershell
> irm get.scoop.sh -outfile 'install.ps1'
> .\install.ps1 -ScoopDir 'D:\Applications\Scoop' -ScoopGlobalDir 'F:\GlobalScoopApps'
```

### 管理软件

### 搜索软件

搜索可安装软件，列出所有符合条件的软件

```powershell
> scoop search name
```

#### 查看已安装软件

列出已安装软件

```powershell
> scoop list
```

#### 查看软件信息

查看软件信息，如主页，描述，最后更新时间

```powershell
> scoop info name
```

#### 安装软件

安装指定名称软件

```powershell
> scoop install nodejs-lts
> node -v
```

#### 更新软件

更新指定软件

```powershell
> scoop update name
> scoop update * # 更新所有软件
```

#### 卸载软件

卸载指定软件，执行卸载脚本(删除环境变量等)

```powershell
> scoop uninstall name
```

#### 清理软件

软件更新后旧版本并不会被卸载，执行命令进行清理

```powershell
> scoop uninstall *
```

### 桶管理

桶(bucket)保存着 Scoop 的可安装软件信息。安装完毕后只有一个桶 - main

#### 搜索桶

只能在 web 中搜索[可用桶](https://scoop.sh/#/buckets)

#### 列出官方认证的桶

```powershell
> scoop bucket known
```

#### 列出已添加桶

```powershell
> scoop bucket list
```

#### 添加桶

```powershell
> scoop bucket add extras # 含 GUI 的软件
> scoop bucket add versions # 旧版软件
> scoop bucket add nerd-fonts # 字体
> scoop bucket add nonportable # 非便携软件
> scoop bucket add java # 各种jdk
> scoop bucket add dorado https://github.com/chawyehsu/dorado # 半官方性质的桶
```

#### 删除桶

```powershell
> scoop bucket rm name
```

## 进阶用法

### 优化搜索速度

scoop 的 search 命令会搜索远程服务器，而 [scoop-search](https://github.com/shilangyu/scoop-search) 只搜索本地桶内容

```powershell
> scoop install scoop-search
> scoop-search name
```

### 软件重名

如果软件重名，使用如下方式安装

```powershell
> scoop install bucket/name
```

### 指定软件版本

如果要安装旧版软件且 versions 中没有，使用如下方式安装

```powershell
> scoop install name@version
```

一个该软件的 manifest 将会被生成在 `~\scoop\workspace`，该软件将不会被更新

### 多版本软件优先级

如果你安装了一个软件的多个衍生版本，例如 temurin-lts-jdk 和 temurin8-jdk，最后一个安装的版本将会在命令行生效，如果需要在命令行使用其他版本，执行

```powershell
> scoop reset name
```
