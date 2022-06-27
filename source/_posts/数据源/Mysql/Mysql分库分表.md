---
title: Mysql分库分表
date: {{ date }}
categories:
- 数据源
- Mysql
---

## 分表方式

垂直分表：根据业务分表

水平分表：根据数据分表

## 分库分表的问题

1. 跨节点连接查询问题（分页、排序）：两张表存在不同的库中，需要进行多次查询。
2. 多数据源管理问题：一个服务中可能包含多个数据源。

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

