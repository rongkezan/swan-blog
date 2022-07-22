---
title: Redis漏斗算法
date: {{ date }}
categories:
- Redis
---

# redis漏斗算法

漏洞的容量是有限的，如果将漏嘴堵住，然后一直往里面灌水，它就会变满，直至再也装不进去。如果将漏嘴放开，水就会往下流，流走一部分之后，就又可以继续往里面灌水。如果漏嘴流水的速率大于灌水的速率，那么漏斗永远都装不满。如果漏嘴流水速率小于灌水的速率，那么一旦漏斗满了，灌水就需要暂停并等待漏斗腾空。

## Java 实现漏斗算法

```java
/**
 * 漏斗算法
 *
 * Funnel 对象的 make_space 方法是漏斗算法的核心，其在每次灌水前都会被调用以触发
 * 漏水，给漏斗腾出空间来。能腾出多少空间取决于过去了多久以及流水的速率。Funnel 对象
 * 占据的空间大小不再和行为的频率成正比，它的空间占用是一个常量。
 *
 * 问题来了，分布式的漏斗算法该如何实现？能不能使用 Redis 的基础数据结构来搞定？
 * 我们观察 Funnel 对象的几个字段，我们发现可以将 Funnel 对象的内容按字段存储到一
 * 个 hash 结构中，灌水的时候将 hash 结构的字段取出来进行逻辑运算后，再将新值回填到
 * hash 结构中就完成了一次行为频度的检测。
 *
 * 但是有个问题，我们无法保证整个过程的原子性。从 hash 结构中取值，然后在内存里
 * 运算，再回填到 hash 结构，这三个过程无法原子化，意味着需要进行适当的加锁控制。而
 * 一旦加锁，就意味着会有加锁失败，加锁失败就需要选择重试或者放弃。
 *
 * 如果重试的话，就会导致性能下降。如果放弃的话，就会影响用户体验。同时，代码的
 * 复杂度也跟着升高很多。这真是个艰难的选择，我们该如何解决这个问题呢？Redis-Cell 救
 * 星来了！
 */
public class FunnelRateLimiter {
    
    private final Map<String, Funnel> funnels = new HashMap<>();

    public boolean isActionAllowed(String userId, String actionKey, int capacity, float leakingRate) {
        String key = String.format("%s:%s", userId, actionKey);
        Funnel funnel = funnels.get(key);
        if (funnel == null) {
            funnel = new Funnel(capacity, leakingRate);
            funnels.put(key, funnel);
        }
        return funnel.watering(1); // 需要 1 个 quota
    }

    static class Funnel {
        private int capacity;
        private float leakingRate;
        private int leftQuota;
        private long leakingTs;

        public Funnel(int capacity, float leakingRate) {
            this.capacity = capacity;
            this.leakingRate = leakingRate;
            this.leftQuota = capacity;
            this.leakingTs = System.currentTimeMillis();
        }

        void makeSpace() {
            long nowTs = System.currentTimeMillis();
            long deltaTs = nowTs - leakingTs;
            int deltaQuota = (int) (deltaTs * leakingRate);
            if (deltaQuota < 0) { // 间隔时间太长，整数数字过大溢出
                this.leftQuota = capacity;
                this.leakingTs = nowTs;
                return;
            }
            if (deltaQuota < 1) { // 腾出空间太小，最小单位是 1
                return;
            }
            this.leftQuota += deltaQuota;
            this.leakingTs = nowTs;
            if (this.leftQuota > this.capacity) {
                this.leftQuota = this.capacity;
            }
        }

        boolean watering(int quota) {
            makeSpace();
            if (this.leftQuota >= quota) {
                this.leftQuota -= quota;
                return true;
            }
            return false;
        }
    }
}
```

## Redis-Cell

> Redis 4.0 提供了一个限流 Redis 模块，它叫 redis-cell。该模块也使用了漏斗算法，并提供了原子的限流指令。有了这个模块，限流问题就非常简单了。

该模块只有 1 条指令 cl.throttle

```shell
> cl.throttle laoqian:reply 15 30 60 1
--------------------------------------
1. (integer) 0	# 0表示允许 1表示拒绝
2. (integer) 15 # 漏斗容量 capacity
3. (integer) 14 # 漏斗剩余空间 left_quota
4. (integer) -1 # 如果拒绝了，需要多长时间后再试(秒)
5. (integer) 2	# 多长时间后，漏斗完全空出来(left_quota == capacity，单位秒)
```

这个指令的意思是允许「用户老钱回复行为」的频率为每 60s 最多 30 次(漏水速率)，漏斗的初始容量为 15，也就是说一开始可以连续回复 15 个帖子，然后才开始受漏水速率的影响。我们看到这个指令中漏水速率变成了 2 个参数，替代了之前的单个浮点数。用两个参数相除的结果来表达漏水速率相对单个浮点数要更加直观一些。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210205164104345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)