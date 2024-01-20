---
title: 重装系统后恢复Scoop
date: 2023-04-09 13:16:35
tags:
---

近期重装了系统，之前 scoop 使用安装的软件不想花时间重装，搜索到以下答案：

- [How to use scoop after reinstalling the system][issuecomment]
- [重装系统后如何恢复使用 scoop][blog]

一顿操作完成后，可以正常使用软件，但是在更新时会报如下错误，原因是 git 默认拒绝解析非当前用户拥有的存储库的配置

```
fatal: detected dubious ownership in repository
```

一个解决办法是执行如下命令

```
git config --global --add safe.directory path/to/error/directory
```

如果 bucket 文件夹较多，可以执行

```
git config --global --add safe.directory *
```

该方法有一定的安全风险，另一种从根源解决的方法为

```
更改scoop文件夹所有者为当前用户，并勾选应用到所有的子目录和文件
```

[issuecomment]: https://github.com/ScoopInstaller/Scoop/issues/2894#issuecomment-447276027
[blog]: https://jiayaoo3o.github.io/2019/03/19/%E9%87%8D%E8%A3%85%E7%B3%BB%E7%BB%9F%E5%90%8E%E5%A6%82%E4%BD%95%E6%81%A2%E5%A4%8D%E4%BD%BF%E7%94%A8scoop/
