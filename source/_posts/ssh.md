---
title: 在 Windows 下愉快地使用 SSH
date: 2023-04-01 19:53:25
tags: [SSH, GPG, 服务器]
categories: [配置]
---

Windows 现在自带OpenSSH客户端，配合git进行使用

<!-- more -->

## 生成 SSH key

使用 `Ed25519` 算法生成 SSH key

```bash
> ssh-keygen -t ed25519 -C "your_email@example.com"
```

确认生成路径

```bash
Enter file in which to save the key (C:\Users\you/.ssh/id_ed25519):
```

输入两次 passphrase，可以为空，为空时一旦有人获取私钥，则可以访问所有配置了该私钥的系统，不建议为空。

```bash
Enter passphrase (empty for no passphrase):

Enter same passphrase again:

Your identification has been saved in C:\Users\you/.ssh/id_ed25519

Your public key has been saved in C:\Users\you/.ssh/id_ed25519.pub
```

生成后相应目录会产生两个文件，`.pub`后缀为公钥，另一个为私钥。

## 使用 SSH Key

使用方式以`GitHub`为例，进入`Settings`->`SSH and GPG keys`->`New SSH key`，将公钥内容粘贴进`Key`框中，点击`Add SSH key`，

测试 github 认证

```bash
> ssh -T git@github.com

Enter passphrase for key 'C:\Users\you/.ssh/id_ed25519':

Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
```

## 配置 ssh-agent

在做完以上步骤后， SHH Key 已经可以正常使用了，但是每次都要输入 passphrase，十分麻烦，可以利用 ssh-agent 对私钥解密结果进行缓存。

使用键盘输入 `Win` + `R`，输入`services.msc`进入服务。打开`OpenSSH Authentication Agent`项目，启动类型选择自动或自动(延迟启动)，点击启动。

添加 SSH key 到 ssh-agent

```bash
> ssh-add

Enter passphrase for C:\Users\lusha/.ssh/id_ed25519:

Identity added: C:\Users\you/.ssh/id_ed25519 (your_email@example.com)
```

操作完成后，删除私钥文件。建议将私钥文件加密备份，以防操作系统重装等因素导致私钥丢失

## GPG

```bash
$ gpg --full-generate-key
gpg (GnuPG) 2.2.29-unknown; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: directory '/c/Users/lusha/.gnupg' created
gpg: keybox '/c/Users/lusha/.gnupg/pubring.kbx' created
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
  (14) Existing key from card
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: lsq27
Email address: lushaoqiang27@gmail.com
Comment:
You selected this USER-ID:
    "lsq27 <lushaoqiang27@gmail.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: /c/Users/lusha/.gnupg/trustdb.gpg: trustdb created
gpg: key 14F807AABAA53E8E marked as ultimately trusted
gpg: directory '/c/Users/lusha/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/c/Users/lusha/.gnupg/openpgp-revocs.d/99A618713B52EF6F70AE461A14F807AABAA53E8E.rev'
public and secret key created and signed.

pub   rsa4096 2023-03-31 [SC]
      99A618713B52EF6F70AE461A14F807AABAA53E8E
uid                      lsq27 <lushaoqiang27@gmail.com>
sub   rsa4096 2023-03-31 [E]
```

---

> 转载请注明出处，使用时须遵循 [CC BY-SA 4.0 License](https://creativecommons.org/licenses/by-sa/4.0/deed.zh) 中所述条款。

# 在 Windows 下愉快地使用 SSH

## 生成 SSH key

使用 `Ed25519` 算法生成 SSH key

```bash
> ssh-keygen -t ed25519 -C "your_email@example.com"
```

确认生成路径

```bash
Enter file in which to save the key (C:\Users\you/.ssh/id_ed25519):
```

输入两次 passphrase，可以为空，为空时一旦有人获取私钥，则可以访问所有配置了该私钥的系统，不建议为空。

```bash
Enter passphrase (empty for no passphrase):

Enter same passphrase again:

Your identification has been saved in C:\Users\you/.ssh/id_ed25519

Your public key has been saved in C:\Users\you/.ssh/id_ed25519.pub
```

生成后相应目录会产生两个文件，`.pub`后缀为公钥，另一个为私钥。

## 使用 SSH Key

使用方式以`GitHub`为例，进入`Settings`->`SSH and GPG keys`->`New SSH key`，将公钥内容粘贴进`Key`框中，点击`Add SSH key`，

测试 github 认证

```bash
> ssh -T git@github.com

Enter passphrase for key 'C:\Users\you/.ssh/id_ed25519':

Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
```

## 配置 ssh-agent

在做完以上步骤后， SHH Key 已经可以正常使用了，但是每次都要输入 passphrase，十分麻烦，可以利用 ssh-agent 对私钥解密结果进行缓存。

使用键盘输入 `Win` + `R`，输入`services.msc`进入服务。打开`OpenSSH Authentication Agent`项目，启动类型选择自动或自动(延迟启动)，点击启动。

添加 SSH key 到 ssh-agent

```bash
> ssh-add

Enter passphrase for C:\Users\lusha/.ssh/id_ed25519:

Identity added: C:\Users\you/.ssh/id_ed25519 (your_email@example.com)
```

操作完成后，删除私钥文件。建议将私钥文件加密备份，以防操作系统重装等因素导致私钥丢失

## GPG

```bash
$ gpg --full-generate-key
gpg (GnuPG) 2.2.29-unknown; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: directory '/c/Users/lusha/.gnupg' created
gpg: keybox '/c/Users/lusha/.gnupg/pubring.kbx' created
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
  (14) Existing key from card
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: lsq27
Email address: lushaoqiang27@gmail.com
Comment:
You selected this USER-ID:
    "lsq27 <lushaoqiang27@gmail.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: /c/Users/lusha/.gnupg/trustdb.gpg: trustdb created
gpg: key 14F807AABAA53E8E marked as ultimately trusted
gpg: directory '/c/Users/lusha/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/c/Users/lusha/.gnupg/openpgp-revocs.d/99A618713B52EF6F70AE461A14F807AABAA53E8E.rev'
public and secret key created and signed.

pub   rsa4096 2023-03-31 [SC]
      99A618713B52EF6F70AE461A14F807AABAA53E8E
uid                      lsq27 <lushaoqiang27@gmail.com>
sub   rsa4096 2023-03-31 [E]
```

---

> 转载请注明出处，使用时须遵循 [CC BY-SA 4.0 License](https://creativecommons.org/licenses/by-sa/4.0/deed.zh) 中所述条款。
