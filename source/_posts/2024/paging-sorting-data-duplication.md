---
title: 数据库分页排序数据重复
date: 2024-01-01 19:53:25
tags:
  - 数据库
  - 生产问题
categories:
  - 解决问题
---

在需求测试中碰到了分页排序数据重复的问题，在翻页时发现部分数据在多页中出现。初次碰到感觉该问题比较反直觉，排序和分页放在一起用时，在我看来应该是先排序再分页，排序顺序确定的情况下，分页每次取部分数据，是不应该出现数据重复出现在不同页的情况的。

<!-- more -->

## 问题复现

开发环境用的是 `Oracle`，在自己的电脑用 `MySQL` 也可以复现问题，`MySQL` 版本采用 8.2.0。

### 准备数据

新建如下表

```sql
create table test(
    id int primary key auto_increment,
    name varchar(10)
);
```

插入如下数据

| id  | name |
| --- | ---- |
| 1   | 1    |
| 2   | 2    |
| 3   | 3    |
| 4   | 3    |
| 5   | 1    |
| 6   | 2    |
| 7   | 3    |
| 8   | 2    |
| 9   | 3    |
| 10  | 1    |

### 分页查询

现在进行分页查询，每页两条数据，查询第一页数据

```sql
select * from test order by name limit 0,2;
```

结果为

| id  | name |
| --- | ---- |
| 1   | 1    |
| 10  | 1    |

查询第二页数据

```sql
select * from test order by name limit 2,2;
```

预期的结果为

| id  | name |
| --- | ---- |
| 5   | 1    |
| x   | 2    |

实际结果如下

| id  | name |
| --- | ---- |
| 10  | 1    |
| 2   | 2    |

可以看到，id 为 10 的数据在两次分页查询中出现了两次，id 为 5 的数据从未出现。通过分页查询，用户永远也无法看到 id 为 5 的数据。

## 原因分析

查询相关资料，MySQL 的文档其实说明了原因
[MySQL :: MySQL 8.3 Reference Manual :: 8.2.1.19 LIMIT Query Optimization](https://dev.mysql.com/doc/refman/8.3/en/limit-optimization.html)

> If multiple rows have identical values in the `ORDER BY` columns, the server is free to return those rows in any order, and may do so differently depending on the overall execution plan. In other words, the sort order of those rows is nondeterministic with respect to the nonordered columns.
>
> One factor that affects the execution plan is `LIMIT`, so an `ORDER BY` query with and without `LIMIT` may return rows in different orders.

简言之，`ORDER BY` 的列如果值相同，数据库不保证返回行的顺序，结果受整体执行计划影响。

那么文章开头问题的原因显而易见，`LIMIT`改变了执行计划，`ORDER BY` 受此影响，对不同的`LIMIT`参数返回了顺序不同的结果行，通过以下`SQL`可以证实

```sql
select * from test order by name limit 0,4;
```

执行结果为

| id  | name |
| --- | ---- |
| 1   | 1    |
| 5   | 1    |
| 10  | 1    |
| 2   | 2    |

从第 3 行起取 2 行结果，与查询第二页数据结果一致。

**`ORDER BY` 的结果顺序受执行计划影响， `LIMIT` 改变了执行计划，导致了分页排序数据重复问题的发生。**

## 解决方案

`MySQL`文档给的解决方案是在`ORDER BY` 的字段中增加不含重复值的列，如`id`

```sql
select * from test order by name, id limit 0,2;
```

结果为

| id  | name |
| --- | ---- |
| 1   | 1    |
| 5   | 1    |

```sql
select * from test order by name, id limit 2,2;
```

结果为

| id  | name |
| --- | ---- |
| 10  | 1    |
| 2   | 2    |

问题解决。

## TODO

执行文中的`SQL`时，执行计划中均出现了 `Using filesort`，之后另起一篇分析一下`MySQL`的排序算法，找到本文问题的代码根源。

| id  | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra          |
| --- | ----------- | ----- | ---------- | ---- | ------------- | --- | ------- | --- | ---- | -------- | -------------- |
| 1   | SIMPLE      | test  |            | ALL  |               |     |         |     | 10   | 100.0    | Using filesort |
