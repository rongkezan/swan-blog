---
title: Mysql锁机制
date: {{ date }}
categories:
- Mysql
---

## 锁概述

Mysql有三种锁：表锁(偏读)、行锁(偏写)、页锁

### 不同的存储引擎支持不同的锁机制

- MyISAM 和 MEMORY 存储引擎采用的是**表级锁**
- InnoDB 存储引擎既支持**行级锁**，也支持**表级锁**，默认情况下是采用**行级锁**

### 锁的是索引

- 表级锁： 开销小，加锁快；不会出现死锁(因为 MyISAM 会一次性获得 SQL 所需的全部锁)。锁定粒度大，发生锁冲突的概率最高，并发度最低。 

- 行级锁： 开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高。 

- 页锁：开销和加锁速度介于表锁和行锁之间；会出现死锁；锁定粒度介于表锁和行锁之间， 并发度一般。

## 查看锁命令

### 查看锁

```sql
show open tables;
```
In_use为0表示没有被锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200201080542825.png)

### 分析表锁定
```sql
show status like '%table%'
```
- Table_locks_immediate：产生表级锁定的次数（锁的查询次数）。
- Table_locks_waited：出现表级锁定争用而发生等待的次数，此值高说明存在严重表级锁争用情况。
## 表锁
### 读锁（共享锁）
Session 1 为Table增加读锁之后：
- Session 1 只能读锁定表，不能读其他表，写锁定表报错。
- Session 2 可以读任何表，写锁定表阻塞。
### 写锁（独占锁）
Session 1 为Table增加写锁之后：
- Session 1 可以做锁定表进行任何操作
- Session 2 无法对锁定表进行任何操作
## 行锁
### 开启事务即开启了行锁
提交事务之前，其它会话查询到的都是未提交的数据，如果更新了同一行，会被阻塞，直到这个事务被提交。
```sql
set autocommit = 0;
update dept set dname = '开发部2' where deptno = 1; 
commit;
```
### 悲观锁
当查询deptno=1的数据的时候，加`for update`语句，此时其它会话修改这条记录就会被阻塞。
```sql
select * from dept where deptno = 1 for update;
```

### 乐观锁

数据表中增加 `version` 字段

- 每次更新时，都需要查询 `version` 的值，符合条件则更新

- 每次更新时，`version` 都在原基础上+1

```sql
select version from t_goods where id = #{id}
update t_goods set status=2, version=version+1 where id=#{id} and version=#{version};  
```

