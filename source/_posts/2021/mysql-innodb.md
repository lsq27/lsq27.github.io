---
title: 读《MySQL技术内幕》
date: 2021-03-21 20:52:31
tags:
---

Spring Boot

<!-- more -->

## 所示

- 数据库：物理操作系统文件或其他形式文件集合
- 实例：是数据库管理软件，由后台线程和一个共享内存区组成

数据库和实例一般一一对应，集群情况下可能一个数据库被多个数据实例使用。

MySQL 启动时会按顺序读取配置文件，一个配置属性出现多次时以最后出现的为准。

```log
Default options are read from the following files in the given order:
C:\Windows\my.ini
C:\Windows\my.cnf
C:\my.ini
C:\my.cnf
C:\Users\lusha\scoop\apps\mysql-lts\current\my.ini
C:\Users\lusha\scoop\apps\mysql-lts\current\my.cnf
```

## MySQL 体系结构

- 连接池
- 管理服务和工具
- SQL 接口组件
- 查询分析器组件
- 优化器组件
- 缓存组件
- 插件式存储引擎
- 物理文件

最重要的特点是插件式的表存储引擎，MySQL 定义了存储引擎接口，用户可以定制开发自己的存储引擎。应该根据具体的应用选择合适的存储引擎。

### 存储引擎

#### InnoDB

最广泛使用的引擎，为面向 OLTP 的应用设计，支持行锁、外键，读操作不加锁。一个表对应一个`ibd`文件。

通过 MVCC 实现高并发，实现了 4 种隔离级别，默认为 REPEATABLE。通过 next-key locking 避免幻读。

采用聚簇方式存储数据，按主键顺序进行存放，如果没有定义主键，引擎会为每行数据生成一个 6 字节的 ROWID 作为主键。

#### MyISAM

不支持事务，表锁设计，支持全文索引，为面向 OLAP 的应用设计。

`myd`存储数据，`myi`存放索引，默认支持 256TB 单表数据。

#### Memory

数据存放在内存中，默认使用哈希索引。MySQL 使用 Memery 作为临时表来存放查询的中间结果集，如果结果集大于设置的容量，或者含有 TEXT 或 BLOB 字段，则使用 MyISAM 引擎。

### 连接方式

一般使用 TCP/IP 套接字，通过网络进行连接；如果客户端和服务器在同一台服务器，在 Windows 下可以使用命名管道或共享内存进行连接，在 Linux/UNIX 下可以使用 UNIX 域套接字进行连接；

## InnoDB 存储引擎

### 后台线程

InnoDB 存储引擎是多线程模型

- Master Thread：负责将缓冲池中的数据异步刷新到磁盘
- IO Thread：负责 AIO 的请求回调，分为 write、read、insert buffer、log 四类线程
- Purge Thread：回收已使用并分配的 undo 页
- Page Cleaner Thread：脏页刷新

### 内存

#### 缓冲池

通过内存的速度弥补磁盘速度较慢对数据库性能的影响。读取磁盘上的页后，将页放在缓存池中，下次直接读取内存的页。修改数据时，首先将页读取到缓冲池，修改缓冲池中的页，以一定的频率将页保存到磁盘上。参数`innodb_buffer_pool_size`决定缓冲池的大小。

缓冲池中缓存的数据页类型有：索引页、数据页、undo 页、插入缓冲、自适应哈希索引、锁信息、数据字典信息等。页大小默认为 16KB，使用优化的 LRU 算法进行管理，最新的页放到尾端 37% 处，前端是 new 区，后端是 old 区，当页在该位置到达 1000ms 后将页放入 new 区首位，这是为了防止扫描大量数据的时候将真正的热点数据挤出缓存。

#### Checkpoint

如果页中的记录被改变了，此时称为页是脏的
