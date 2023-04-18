---
title: Mysql死锁
date: {{ date }}
categories:
- 线上问题
---

## 报错信息

```
Error updating database.  Cause: com.mysql.cj.jdbc.exceptions.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
```

原因：一个用户同时售出了很多藏品，在售出藏品后我会去同步藏品，即先删除旧藏品，再新增新藏品，在并发场景下，会出现死锁。

## 场景复现

### 数据表

InnoDB存储引擎 + 默认的事务隔离级别 Repeatable Read

用MySQL客户端模拟并发事务操作数据时，如下表按照时间的先后顺序执行命令，会导致死锁。

```shell
select * from a ;
+----+
| id |
+----+
|  3 |
+----+
|  8 |
+----+
|  11 |
+----+
```

### 出问题的SQL

```
1 begin;
2 delete from a where id = 4;
3 begin;
4 delete from a where id = 6;
5 insert into a values(5);
6 insert into a values(7);
7 Query OK, 1 row affected
8 ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
9 commit;
```

### 锁的分类

- InnoDB存储引擎实现共享锁（S Lock）和排它锁（X Lock）两种行级锁。
  - S Lock：允许事务读一行数据，多个事务可以并发的对行数据加S Lock
  - X Lock：允许事务删除或更新一行数据，只有行数据没有任何锁才可以获取X Lock
- InnoDB支持意向共享锁（IS Lock）和意向排它锁（IX Lock），这两种锁是表级别的锁，但实际上也应用在行锁之中
  - IS Lock：事务想要获得一张表中某几行的共享锁
  - IX Lock：事务想要获得一张表中某几行的排它锁

- 行锁：锁定一行数据，即上面所说的共享锁和排他锁
- 间隙锁：锁定一个范围，但不包含记录本身。例如数据库中数据id为3,8,11，那么锁定的区间可能为(-∞, 3) (3, 8) (8, 11)，(11,+∞)，假如插入的数据为6，那么区间(3,8)被锁定，但不包括6
- 行锁 + 间隙锁：锁定一个范围，包括记录本身，例如区间(3,8)被锁定时，要插入的数据6也会被锁定

为什么要有间隙锁？

我们应该听说过幻读，即在同一事务下，连续执行两次同样的SQL语句可能导致不同的结果，第二次的SQL语句可能返回之前不存在的行。InnoDB使用行锁 + 间隙锁的方式解决这个问题。当然，InnoDB存储引擎在查询数据时是不存在锁的，这是因为查询的数据来自于快照版本，即历史数据。

### 数据库锁的应用

insert 插入记录时，需要获取行锁

update 更新一条记录时，如果记录存在，需要行锁；如果记录不存在，行锁 + 间隙锁

delete 删除一条记录时，如果记录存在，需要行锁；如果记录不存在，行锁 + 间隙锁

select 查询记录时，不会存在锁，除非显示的调用lock in share mode或者for update，如下所示。为什么查询不存在锁呢？因为InnoDB引擎select查询返回的是数据的快照版本，这也是为什么在许多mysql书中，事务的select查询需要锁时，要显式的使用加锁语法

```sql
-- S Lock
select * from a where id = 1 lock in share mode;
-- X Lock
select * from a where id = 1 for update ;
```

### 原因分析

为什么看似互不影响的事务会出现死锁的问题？

上面发生死锁的情况是当数据不存在时，当数据存在时，也会出现死锁的情况，这种情况可以通过3个会话来模拟，当然在实际的项目情况下，并发事务确实是带来了死锁的问题，例如在Spring事务中，先删除表A中的数据，再向表A插入数据，如果并发量比较大的话，如果存在间隙锁，那么有几率会出现死锁的问题。

Spring事务中大致的运行流程如下：

一个事务中存在先删除在插入的逻辑，并发时，事务A将存在的数据id=6删除，此时事务B也删除id=6的数据，事务C同样删除id=6的数据，这种情况下，如果并发量够大，一定会出现间隙锁，从而发生死锁。

### 如何避免

如何去避免？通常情况下，要删除一条数据，那么需要先查询数据是否存在，如果存在，再去删除，否则不执行删除逻辑。其实这种方式也存在一定的风险，我们可以通过软删除的方式，避免高并发时出现数据已被删除，而其他事务正在删除不存在的数据。软删除是指通过字段决定数据是否已删除，然后定时的手动处理数据库中的数据。

