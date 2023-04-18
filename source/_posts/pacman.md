---
title: pacman
date: 2023-04-08 19:59:51
tags: [包管理, Linux, Arch]
---

最近有在 Windows 下编译 C++的需要，不想使用笨重的 Visual Studio，于是采用 MingGW，考虑到可能的包管理器需求，选择 MSYS2。MSYS2 的包管理采用的是来自 Arch Linux 的
pacman。它将简单的二进制包格式与易于使用的构建系统相结合。pacman 的目标是让轻松管理软件包成为可能，无论它们是来自官方仓库还是用户自己的构建。

Pacman 只需一个命令即可更新系统上的所有软件包。这可能需要相当长的时间，具体取决于系统的最新程度。以下命令同步存储库数据库并更新系统的软件包，不包括不在已配置存储库中的“本地”软件包：

```bash
pacman -Syu
```

安装软件包

```bash
pacman -Ss string1 string2 ...
```

要搜索已安装的软件包，请执行以下操作：

```bash
pacman -Qs string1 string2 ...
```

查看包的依赖关系树：

```bash
pactree package_name
```

安装软件包

```bash
pacman -S package_name1 package_name2 ...
```

安装不是来自远程仓库的“本地”软件包（例如，该软件包来自 AUR）：

```bash
pacman -U /path/to/package/package_name-version.pkg.tar.zst
```

要删除单个包，保留其所有依赖项，请执行以下操作：

```bash
pacman -R package_name
```

要删除任何其他已安装软件包不需要的软件包及其依赖项，请执行以下操作：

```bash
pacman -Rs package_name
```

paccache（8） 脚本， 在 pacman-contrib 软件包中提供， 默认情况下会删除所有已安装和卸载的软件包的缓存版本， 除了最近的三个：

```bash
paccache -r
```

Pacman 通过与主服务器同步软件包列表来保持系统最新状态。此服务器/客户端模型还允许用户使用简单的命令下载/安装包，并完成所有必需的依赖项。

Pacman 是用 C 编程语言编写的， 使用 bsdtar（1） tar 格式进行打包。
