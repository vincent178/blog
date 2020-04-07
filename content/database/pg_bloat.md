---
title: "深入浅出 PG 数据膨胀"
date: 2020-03-29T07:41:53+08:00
draft: true
---

## 多版本并发控制 MVCC

多版本并发控制，是数据库管理系统常用的一种并发控制，每个事务读到的数据项都是一个历史快照（snapshot）并依赖于实现的隔离级别。写操作不覆盖已有数据项，而是创建一个新的版本，直至所在操作提交时才变为可见。Postgres（以下简称PG）中的多版本，是保存在各个表中的，相比较 MySQL 是保存在全局中的。

### 隐藏的版本信息

PG 各个表独立保存着多版本信息，也意味着在每条记录都有版本号。通过一些技巧可以挖掘到这些隐藏的信息。

```
pg_ninja=# \d employee
           Table "public.employee"
 Column  |         Type          | Modifiers
---------+-----------------------+-----------
 emp_id  | integer               | not null
 name    | character varying(20) |
 dept_id | integer               |
Indexes:
    "employee_pkey" PRIMARY KEY, btree (emp_id)
```

在 `pg_attribute` 中可以看到几个隐藏的字段：
```
pg_ninja=# SELECT attname, format_type (atttypid, atttypmod)
pg_ninja-# FROM pg_attribute
pg_ninja-# WHERE attrelid::regclass::text='employee'
pg_ninja-# ORDER BY attnum;
 attname  |      format_type
----------+-----------------------
 tableoid | oid
 cmax     | cid
 xmax     | xid
 cmin     | cid
 xmin     | xid
 ctid     | tid
 emp_id   | integer
 name     | character varying(20)
 dept_id  | integer
(9 rows)
```

通过 `SELECT` 返回隐藏字段的值：
```
pg_ninja=# SELECT txid_current();
 txid_current
--------------
      1088042
(1 row)

pg_ninja=# INSERT into employee VALUES (9,'avi',9);
INSERT 0 1
pg_ninja=# SELECT xmin,xmax,* FROM employee where emp_id = 9;
  xmin   | xmax | emp_id | name | dept_id
---------+------+--------+------+---------
 1088043 |    0 |      9 | avi  |       9
(1 row)
```

xmin：记录了创建（INSERT）记录的事务 ID（简称 xid）。

通过 `SELECT txid_current()` 可以得到当前的 xid，下一个 INSERT 的 xid 是当前 `xid+1` 就是 1088043，与保存在 `xmin` 中的值相等。

xmax：记录了删除（DELETE）记录的 xid。

删除之后该记录就不能简单的 SELECT 返回，这里通过两个会话在验证 xmax 的值。

在终端 A 中开始事务，删除但不提交。

```
pg_ninja=# begin;
BEGIN
pg_ninja=# SELECT txid_current();
 txid_current
--------------
      1088046
(1 row)

pg_ninja=# DELETE FROM employee where emp_id = 9;
DELETE 1
```

在终端 B 中连接数据库，查询 xmax 值。
```
pg_ninja=# SELECT xmin, xmax, * FROM employee where emp_id = 9;
  xmin   |  xmax   | emp_id | name | dept_id
---------+---------+--------+------+---------
 1088043 | 1088046 |      9 | avi  |       9
(1 row)
```

因为在一个事务中，`SELECT txid_current()` 就是删除语句的 xid 即 1088046，与保存在 `xmax` 中的值相等。

### “删除” 的记录

删除的记录在 PG 中并没有真正的删除。通过 pageinspect 模块可以窥视 PG 底层的存储秘密。

```
pg_ninja=# CREATE EXTENSION pageinspect;
pg_ninja=# INSERT into employee values (generate_series(1,10),'avi',1);
INSERT 0 10
pg_ninja=# SELECT t_xmin, t_xmax, t_data FROM heap_page_items(get_raw_page('employee', 0));
 t_xmin  | t_xmax |           t_data
---------+--------+----------------------------
 1089069 |      0 | \x010000000961766901000000
 1089069 |      0 | \x020000000961766901000000
 1089069 |      0 | \x030000000961766901000000
 1089069 |      0 | \x040000000961766901000000
 1089069 |      0 | \x050000000961766901000000
 1089069 |      0 | \x060000000961766901000000
 1089069 |      0 | \x070000000961766901000000
 1089069 |      0 | \x080000000961766901000000
 1089069 |      0 | \x090000000961766901000000
 1089069 |      0 | \x0a0000000961766901000000
(10 rows)
```

删除其中的五条记录。
```
pg_ninja=# DELETE FROM employee where emp_id  > 5;
DELETE 5
pg_ninja=# SELECT t_xmin, t_xmax, t_data FROM heap_page_items(get_raw_page('employee', 0));
 t_xmin  | t_xmax  |           t_data
---------+---------+----------------------------
 1089069 |       0 | \x010000000961766901000000
 1089069 |       0 | \x020000000961766901000000
 1089069 |       0 | \x030000000961766901000000
 1089069 |       0 | \x040000000961766901000000
 1089069 |       0 | \x050000000961766901000000
 1089069 | 1089147 | \x060000000961766901000000
 1089069 | 1089147 | \x070000000961766901000000
 1089069 | 1089147 | \x080000000961766901000000
 1089069 | 1089147 | \x090000000961766901000000
 1089069 | 1089147 | \x0a0000000961766901000000
(10 rows)
```
可以看到删除的记录依然在底层存储中。

更新剩下的五条记录。
```
pg_ninja=# UPDATE employee SET emp_name = 'avii';
UPDATE 5
pg_ninja=# SELECT t_xmin, t_xmax, t_data FROM heap_page_items(get_raw_page('employee', 0));
 t_xmin  | t_xmax  |               t_data
---------+---------+------------------------------------
 1089069 | 1089148 | \x010000000961766901000000
 1089069 | 1089148 | \x020000000961766901000000
 1089069 | 1089148 | \x030000000961766901000000
 1089069 | 1089148 | \x040000000961766901000000
 1089069 | 1089148 | \x050000000961766901000000
 1089069 | 1089147 | \x060000000961766901000000
 1089069 | 1089147 | \x070000000961766901000000
 1089069 | 1089147 | \x080000000961766901000000
 1089069 | 1089147 | \x090000000961766901000000
 1089069 | 1089147 | \x0a0000000961766901000000
 1089148 |       0 | \x010000000b6176696900000001000000
 1089148 |       0 | \x020000000b6176696900000001000000
 1089148 |       0 | \x030000000b6176696900000001000000
 1089148 |       0 | \x040000000b6176696900000001000000
 1089148 |       0 | \x050000000b6176696900000001000000
(15 rows)
```
PG 更新的逻辑是删除并创建，可以看到这里返回了15条记录，其中10条记录了 `t_xmax`（其中5条是更新时删除的），另外新创建的5条记录的 `t_xmin` 值是新的 xid，而 `t_xmax` 值为 0。

## Vacuum

很显然，维护这样的版本信息的存储成本是巨大的，尤其是更新频繁的表。PG 解决方案就是 Vacuum。这些删除的数据在 PG 中叫做 dead tuple。一旦没有任何事务使用这些数据，这些数据就可以安全的被删除。Vacuum 就是承担了清理这些数据的工作。

## 从库的热反馈 (Hot Standby Feedback 以下简称 HSF)
在配置了主从后，如果打开了 HSF，HSF 会将当前从库的最小 xid 发给主库，目的是解决同步冲突问题。如果从库在使用某些记录，即使在主库中已经删除，并且主库中没有任何事务使用该记录，Vacuum 依然会保留该记录。

Reference:

* https://www.percona.com/blog/2018/08/06/basic-understanding-bloat-vacuum-postgresql-mvcc/
* https://www.postgresql.org/docs/9.6/routine-vacuuming.html
