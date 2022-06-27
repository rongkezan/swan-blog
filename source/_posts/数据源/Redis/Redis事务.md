---
title: Redis事务
date: {{ date }}
categories:
- 数据源
- Redis
---

## 1.简介

redis事务不保证原子性，redis同一个事务中如果有一条命令执行失败，其它命令仍然会被执行，不会回滚。

谁的事务先到达（EXEC）就执行谁的事务

常用命令

| 命令 | 序号         | 描述                                                         |
| ---- | ------------ | ------------------------------------------------------------ |
| 1    | DISCARD      | 取消事务，放弃执行事务块内的所有命令                         |
| 2    | EXEC         | 执行事务块内的命令                                           |
| 3    | MULTI        | 标记一个事务的开始                                           |
| 4    | WATCH key... | 监视一个或多个key，如果事务执行前这些key被其它命令改动，那么事务将被打断 |
| 5    | UNWATCH      | 取消对所有Key的监视                                       


## 2.命令演示
初始化数据，并使用watch监视name
```
127.0.0.1:6379> set name a
OK
```
客户端A开watch监视name
```
127.0.0.1:6379> watch name
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set name aa
QUEUED
```
此时客户端B抢先提交了
```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set name bb
QUEUED
127.0.0.1:6379> exec
1) OK
```
那么客户端A提交会失败
```
127.0.0.1:6379> exec
(nil)
```

### 3.代码演示

```java
while(true){
    stringRedisTemplate.watch(REDIS_LOCK);
    if (uuid.equals(stringRedisTemplate.opsForValue().get(REDIS_LOCK))){
        stringRedisTemplate.setEnableTransactionSupport(true);
        stringRedisTemplate.multi();
        stringRedisTemplate.delete(REDIS_LOCK);
        List<Object> exec = stringRedisTemplate.exec();
        if (CollectionUtils.isEmpty(exec)) continue;
    }
    stringRedisTemplate.unwatch();
    break;
}
```