---
title: Mysql事务
date: {{ date }}
categories:
- 数据源
- Mysql
---

## ACID

原子性（atomicity）：一个事务要么全部提交成功，要么全部失败回滚

一致性（consistency）：一个事务在执行之前和执行之后，数据库都必须处于一致性状态

隔离性（isolation）：一个事务的执行不能不被其他事务干扰

持久性（durability）：一旦事务提交，那么它对数据库中的对应数据的状态的变更就会永久保存到数据库中

## 事务操作

```sql
-- 开启事务
BEGIN;
-- 插入数据
INSERT tb_user VALUES ('张三', 22);
INSERT tb_user VALUES ('李四', 23);
-- 提交事务
COMMIT;
-- 回滚事务
ROLLBACK;

-- 查看自动提交
SHOW VARIABLES LIKE '%autocommit%'
-- 关闭自动提交，只针对当前的会话有效
SET autocommit = 0

-- 设置保存点
SAVEPOINT [pointName];
-- 回滚到指定保存点
ROLLBACK to [pointName];
-- 删除保存点
RELEASE SAVEPOINT [pointName];

-- 查看隔离级别
select @@transaction_isolation;
-- 修改隔离级别
set session transaction isolation level 事务隔离级别;
```
## 隔离级别

对于同时运行多个事务，这些事务访问**数据库中相同的数据**时，如果没有采取必要的隔离机制，就会导致各种并发问题：
- 脏读：事务A读到了事务B已修改但尚未提交的数据，此时事务B回滚，A读取数据无效，不符合一致性。
- 不可重复读：事务A读到了事务B已提交的修改数据，不符合隔离性。
- 幻读：事务A读到了事务B提交的新增数据，不符合隔离性。

| 隔离级别                                  | 脏读 | 不可重复读 | 幻读 |
| ----------------------------------------- | ---- | ---------- | ---- |
| 读未提交 READ UNCOMMITTED                 | 是   | 是         | 是   |
| 读已提交 READ COMMITTED                   | 否   | 是         | 是   |
| 可重复读 REPEATABLE READ（Mysql默认级别） | 否   | 否         | 是   |
| 可序列化 SERIALIZABLE                     | 否   | 否         | 否   |

## MVCC 多版本并发控制

MVCC在MySQL InnoDB中的实现主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读。

- 当前读
  像select lock in share mode(共享锁), select for update ; update, insert ,delete(排他锁)这些操作都是一种当前读，为什么叫当前读？就是它读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。

- 快照读
  像不加锁的select操作就是快照读，即不加锁的非阻塞读；快照读的前提是隔离级别不是串行级别，串行级别下的快照读会退化成当前读；之所以出现快照读的情况，是基于提高并发性能的考虑，快照读的实现是基于多版本并发控制，即MVCC,可以认为MVCC是行锁的一个变种，但它在很多情况下，避免了加锁操作，降低了开销；既然是基于多版本，即快照读可能读到的并不一定是数据的最新版本，而有可能是之前的历史版本

### 实现原理

#### Undo Log

每行记录除了我们自定义的字段外，还有数据库隐式定义的DB_TRX_ID,DB_ROLL_PTR,DB_ROW_ID等字段

- DB_ROW_ID：隐含的自增ID（隐藏主键），如果数据表没有主键，InnoDB会自动以DB_ROW_ID产生一个聚簇索引
- DB_TRX_ID：最近 修改/插入 事务ID，记录创建这条记录/最后一次修改该记录的事务ID
- DB_ROLL_PTR：回滚指针，指向这条记录的上一个版本（存储于rollback segment里）

每次事务提交都会有将其加入 undo log 中，undo log会成为一条记录版本线性表

- 在事务2修改该行数据时，数据库也先为该行加锁
- 然后把该行数据拷贝到undo log中，作为旧记录，发现该行记录已经有undo log了，那么最新的旧数据作为链表的表头，插在该行记录的undo log最前面
- 修改该行age为30岁，并且修改隐藏字段的事务ID为当前事务2的ID, 那就是2，回滚指针指向刚刚拷贝到undo log的副本记录
- 事务提交，释放锁

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210125144344982.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

#### Read View

Read View 就是事务进行快照读操作的时候生产的读视图，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID (当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大)

Read View 遵循一个可见性算法，主要是将要被修改的数据的最新记录中的DB_TRX_ID（当前事务ID）取出来，与系统当前其他活跃事务的ID去对比，如果 DB_TRX_ID 跟 Read View 的属性做了某些比较，不符合可见性，那就通过 DB_ROLL_PTR 回滚指针去取出 Undo Log 中的 DB_TRX_ID 再比较，即遍历链表的DB_TRX_ID（从链首到链尾，即从最近的一次修改查起），直到找到满足特定条件的DB_TRX_ID, 那么这个DB_TRX_ID所在的旧记录就是当前事务能看见的最新老版本