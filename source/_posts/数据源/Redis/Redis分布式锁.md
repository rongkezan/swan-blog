---
title: Redis分布式锁
date: {{ date }}
categories:
- 数据源
- Redis
---

https://juejin.cn/post/6844904106461495303

## 1. 实现方式

redis与zookeeper的区别：redis是ap不保证一致性，zk是cp不保证高可用

redis分布式锁流程：抢锁 - 执行业务 - 删锁

使用redis作分布式锁需要注意的问题：

1. 锁需要设置过期时间，以便异常情况下未能成功删锁
2. 对于加锁和解锁需要保证操作的原子性
3. 删锁时使用uuid作为value判断value是否一致，防止删除别人的锁

```java
@GetMapping("buy")
public String buyGoods() throws Exception {
    String uuid = UUID.randomUUID().toString();
    Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, uuid, 10L, TimeUnit.SECONDS);
    if (flag == null || !flag) return "抢锁失败";
    try {
        // 执行业务逻辑
        return doBuyGoods();
    } finally {
        Jedis jedis = RedisUtils.getJedis();
        String script = "if redis.call(\"get\",KEYS[1]) == ARGV[1]\n" +
                "then\n" +
                "    return redis.call(\"del\",KEYS[1])\n" +
                "else\n" +
                "    return 0\n" +
                "end";
        try {
            Object o = jedis.eval(script, Collections.singletonList(REDIS_LOCK), Collections.singletonList(uuid));
            if ("1".equals(o.toString())){
                System.out.println("delete redis lock ok");
            } else {
                System.out.println("delete redis lock error");
            }
        } finally {
            if (jedis != null){
                jedis.close();
            }
        }
    }
}
```

## 2. redisson

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.14.0</version>
</dependency>
```

```java
@Configuration
public class RedisConfig {
    @Bean
    public Redisson redisson(){
        Config config = new Config();
        config.useSingleServer().setAddress("redis//127.0.0.1:6379").setDatabase(0);
        return (Redisson) Redisson.create(config);
    }
}
```

```java
@Resource
private Redisson redisson;

@GetMapping("buy")
public String buyGoods() {
    RLock lock = redisson.getLock("REDIS_LOCK");
    lock.lock();
    Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, uuid, 10L, TimeUnit.SECONDS);
    if (flag == null || !flag) return "抢锁失败";
    try {
        // 执行业务逻辑
        return doBuyGoods();
    } finally {
        if (lock.isLocked() && lock.isHeldByCurrentThread())
            lock.unlock();
    }
}
```

## 3. redisson 看门狗

用于解决锁过期，业务却还没执行完的场景

原理：他是一个后台线程（定时任务），会每隔10秒检查一下，如果客户端还持有锁key，那么就会不断的延长锁key的生存时间。如果服务挂掉了，定时任务自然也就跑不下去了。

**指定 redisson 锁的过期时间**

```java
lock.lock(10,TimeUnit.SECONDS)
```

