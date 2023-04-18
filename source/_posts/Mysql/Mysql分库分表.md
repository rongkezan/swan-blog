---
title: Mysql分库分表
date: {{ date }}
categories:
- Mysql
---

## 分表方式

垂直分表：根据业务分表

水平分表：根据数据分表

## 分表算法

1. 主键ID取模分表
2. 时间范围分表

## 分库分表的问题

### 分布式唯一ID

#### UUID

- 优点：本地生成，性能高
- 缺点：
  - 一般为36位字符串，更占用空间
  - 不适合作为Mysql主键
    - Mysql是聚簇索引，会把相邻主键的数据放在相邻的物理存储位置上，无序性会导致磁盘随机IO、叶分裂等问题
    - 普通索引需要存储主键值，导致B+树变高
  - 基于MAC地址生成的算法可能导致MAC地址泄露

#### 雪花算法

![在这里插入图片描述](https://img-blog.csdnimg.cn/31b98dc293134cc7a6c604dc6afadb41.png)

#### 号段模式

通过一张表来读取ID，但每次读取1个会造成数据库压力很大，所以采用每次读取一批ID存到本地缓存中，当本地缓存的ID使用完了再申请新的号段

![在这里插入图片描述](https://img-blog.csdnimg.cn/f7a6e12e2fc2423b94bb4f625bdad3cb.png)

### 分表后的分页查询

#### 全局查询

假设分了两张表（表1、表2），分页需要查询第二页的5条记录。

则需要把表1和表2的前10条数据都查出来，进行排序后再取第5-9条。

缺点：会随着页数的增加导致查询的数据量增加。

```sql
select * from t_order_1 order by time asc limit 0,10;
select * from t_order_2 order by time asc limit 0,10;
```

#### 禁止跳页查询

例如第一页查询结果：[1664088181，1664088189，1664088219，1664088289，1664088392]

此时SQL如下：

```sql
select * from t_order_1 where time > 1664088392 order by time asc limit 5;
select * from t_order_2 where time > 1664088392 order by time asc limit 5;
```

全查询出来后再将数据在内存里重新排序，取前5条数据。

缺点：只能一页一页依次查询。

#### 使用搜索引擎ES

将数据异构一份存入ES，查询时通过ES进行查询，不直接查询数据库。

## Mycat

MyCat技术原理中最重要的一个动词是“拦截”，它拦截了用户发送过来的SQL语句，首先对SQL语句做了一些特定的分析：如分片分析、路由分析、读写分离分析、缓存分析等，然后将此SQL发往后端的真实数据库，并将返回的结果做适当的处理，最终再返回给用户。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210301090825393.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## Sharding JDBC

> 轻量级的 Java 框架，增强版的 JDBC

```properties
spring.shardingsphere.datasource.names=ds

spring.shardingsphere.datasource.ds.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.ds.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.ds.url=jdbc:mysql://127.0.0.1:3306/tripper?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Shanghai
spring.shardingsphere.datasource.ds.username=root
spring.shardingsphere.datasource.ds.password=root

spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds.t_order$->{1..2}

spring.shardingsphere.sharding.tables.t_order.key-generator.column=id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE

spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order$->{id % 2 + 1}

spring.main.allow-bean-definition-overriding=true
```

