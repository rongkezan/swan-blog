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

## ACID是靠什么保证的

原子性：通过undolog日志保证，它记录了需要回滚的日志信息，事务撤销回滚时撤销已经执行成功的sql

一致性：由其他三大特性保证，程序代码要保证业务上的一致性

隔离性：MVCC

持久性：通过redolog日志保证，mysql修改数据的时候会在redolog中记录一份日志数据，就算数据没有保存成功，只要日志保存成功了，数据依然不会丢失

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

## MVCC

### MVCC 作用

在Mysql InnoDB存储引擎下，READ COMMITED 和 REPEATABLE READ 基于MVCC（多版本并发控制）进行并发事务控制。

MVCC是基于**数据版本**对并发事务进行访问。

![在这里插入图片描述](https://img-blog.csdnimg.cn/120aa7b801364d769b525be1589c85ef.png)

对于事务D

READ COMMITED 结果：张三、张小三

REPEATABLE READ 结果：张三、张三

### Undo Log版本链

那么Mysql是怎么实现不读到已提交的修改数据？通过Undo Log版本链

在版本链中存有 `事务ID` 和 `回滚指针` ，Mysql会确保版本链数据不再被引用后再将版本链删除

<img src="https://img-blog.csdnimg.cn/a00430ad479947c3850879ebe94f8367.png" alt="在这里插入图片描述" style="zoom: 25%;" />

Undo Log版本链作用：

ReadView："快照读"SQL执行时MVCC提取数据的依据

- ReadView是一个数据结构，包含4个字段：
  - m_ids：当前活跃的事务编号集合
  - min_trx_id：最小活跃事务编号
  - max_trx_id：预分配事务编号，当前最大事务编号+1
  - create_trx_id：ReadView创建者的事务编号
- 快照读：最普通的SELECT查询语句
- 当前读：执行下列语句时进行数据读取的方式
  - INSERT、UPDATE、DELETE、SELECT...FOR UPDATE、SELECT...LOCK IN SHARE MODE

#### READ COMMITED

![在这里插入图片描述](https://img-blog.csdnimg.cn/ac58f0b6898048ba9da55663c42ccb8a.png)

版本链数据访问规则：

判断当前事务ID是否等于create_trx_id(4)，成立说明数据就是这个事务自己更改的，可以访问。

判断 trx_id < min_trx_id(2)，成立说明数据已经提交了，可以访问。

判断 trx_id > max_trx_id(5)，成立说明该事务在ReadView生成以后才开启，不允许访问。

判断 min_trx_id(2) <= trx_id <= max_trx_id(5)，成立则在m_ids数据中对比，不存在数据则代表数据是已提交的，可以访问。

因此根据ReadView可以读到1号事务提交的数据 -- 张三。

#### REPEATABLE READ

![在这里插入图片描述](https://img-blog.csdnimg.cn/b12840415d944d6eaf89cbe5ec6befaf.png)

ReadView会复用第一次产生的，因此不会读到事务2修改已提交的数据

**幻读**

- 连续多次快照读，ReadView会产生复用，没有幻读问题
- 当两次快照读之间存在当前读，ReadView会重新生成，导致产生幻读

![在这里插入图片描述](https://img-blog.csdnimg.cn/85f4b70f21294af78d40093f973e459c.png)