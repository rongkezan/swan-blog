---
title: Zookeeper
date: {{ date }}
categories:
- 开发集
---

## 基本概念

zookeeper是一个目录结构，node可以存放数据（1M）

节点分为：持久节点、临时节点（session）、序列节点。

**zookeeper提供了一系列的保证**

顺序性：客户端发送的请求将会被顺序的处理。

原子性：更新要么失败要么成功，没中间状态。

单系统映像：无论客户端连接到哪个服务器，都将看到相同的服务视图。

可靠性：一旦应用更新，它将从那时起持续到客户端覆盖更新。

实时性：系统的客户视图保证特定的时间范围内是最新的。

**应用场景**

配置中心、负载均衡、命名服务、DNS服务、集群管理

## Paxos

> 分布式一致性算法

### Paxos 概念

Paxos 前提：没有拜占庭将军问题，即所有节点都是可信的，Paxos 只有在一个可信的环境中才能成立。

所有的变更都需要通过一个提议，每个提议都有一个编号，这个编号是一直增长的。每个提议都需要超过半数的议员同意才能生效。每个议员只会同意大于当前编号的提议，包括已生效的和未生效的。议会有一个目标：保证所有的议员对于提议都能达成一致的看法。在所有议员中有一个总统，只有总统有权发出提议，如果议员有自己的提议，必须发给总统并由总统来提出。

> 拜占庭将军：存在消息丢失的不可靠信道上试图通过消息传递的方式达到一致性是不可能的。

### ZAB

> 原子广播协议

Zookeeper 如何实现最终一致性

假设一个 Zookeeper 集群中有3台机器，1台 Leader，2台 Follower

1. 客户端将创建节点的请求发送给 Follower
2. Follower 将请求转发给 Leader
3. Leader 在内部生成一个事务ID，并且将 Log 日志写入队列，推送给 Follower
4. 此时若 Follower1 返回了 ok，而 Follower2 没返回，而 Leader 收到 Follower1 的请求后也返回了 ok，根据半数通过原则，Leader 会将写请求放入队列，所以即使 Follower2 没返回 ok，也会同步数据，实现最终一致性

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210130205016592.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 三种角色

Leader：负责处理集群的写请求，并发起投票，只有超过半数的节点同意后才会提交该请求。

Follower：处理读请求，转发写请求给leader，在选举leader时参与投票。

Observer：处理读请求，转发写请求给leader，没投票能力

```shell
# 将一台服务变成 observer
server.3=node03:2888:3888:observer
```

## Linux 安装 Zookeeper

下载 解压 改名

```shell
wget https://mirror.bit.edu.cn/apache/zookeeper/stable/apache-zookeeper-3.5.9-bin.tar.gz
tar xf apache-zookeeper-3.5.9-bin.tar.gz
mv apache-zookeeper-3.5.9-bin/ zookeeper
```

配置环境变量

```shell
vim /etc/profile

export ZOOKEEPER=/root/server/zookeeper
export PATH=$PATH:$ZOOKEEPER/bin

source /etc/profile
```

启动，查看状态，连接客户端

```shell
zkServer.sh start
zkServer.sh status
zkCli.sh
```

## Zookeeper 基本命令

```shell
# 查看节点
ls /
# 创建节点/test,值为value
create /test "value"
# 不会覆盖创建 会创建出 /test0000000001 节点
create -s /test "value2"
# 获取数据
get /test
# 设置数据
set /test "value2"
# 监听节点
watch /test

----
# 概念解释
cZxid = 0x200000002	# 创建的事务ID
mZxid = 0x200000003	# 修改的事务ID
pZxid = 0x200000004 # 子节点创建/删除的事务ID
```

**watch**

客户端1创建了临时节点 `/test/a`

客户端2监听了这个节点 `/test/a`

当客户端1挂掉的时候，`/test/a` 这个节点会有事件（event）产生，这时可以回调客户端2

## Zookeeper集群

配置 zoo.cfg

```shell
server.1=node01:2888:3888
server.2=node02:2888:3888
server.3=node03:2888:3888
```

3888：选主投票用的

2888：leader 接收 write 请求

## 配置文件

复制 zoo_sample.cfg 到 zoo.cfg

```shell
cp zoo_sample.cfg zoo.cfg
```

参数解释

```shell
# 心跳时间（毫秒）
tickTime=2000
# follower连接leader时，初始化有10次心跳机会连接leader，结合tickTime即20s
initLimit=10
# follower同步leader时，同步心跳有5次机会，结合tikeTime即10s
syncLimit=5
# 持久化目录
dataDir=/tmp/zookeeper
# zookeeper的端口
clientPort=2181
# 允许最大的客户端连接数
maxClientCnxns=60
```

## 选举机制

每个 zk 服务器会有 ID，选举的时候首选会选数据最全的 zk，即 事务ID（Zxid）最大的，如果 Zxid 都一样，那么会选 ID 最大的。

目前有5台服务器，每台服务器均没有数据，它们的编号分别是1,2,3,4,5,按编号依次启动，它们的选择举过程如下：

- 服务器1启动，给自己投票，然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，服务器1的状态一直属于Looking(选举状态)。
- 服务器2启动，给自己投票，同时与之前启动的服务器1交换结果，由于服务器2的编号大所以服务器2胜出，但此时投票数没有大于半数，所以两个服务器的状态依然是LOOKING。
- 服务器3启动，给自己投票，同时与之前启动的服务器1,2交换信息，由于服务器3的编号最大所以服务器3胜出，此时投票数正好大于半数，所以服务器3成为领导者，服务器1,2成为小弟。
- 服务器4启动，给自己投票，同时与之前启动的服务器1,2,3交换信息，尽管服务器4的编号大，但之前服务器3已经胜出，所以服务器4只能成为小弟。
- 服务器5启动，后面的逻辑同服务器4成为小弟。

## Zookeeper Java Api

引入 Zookeeper 依赖

```xml
<!-- Zookeeper -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.5.9</version>
</dependency>
```

测试代码

```java
public static void main(String[] args) throws Exception {
    CountDownLatch latch = new CountDownLatch(1);
    // 连接zk服务
    ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 3000, watchedEvent -> {
        Watcher.Event.KeeperState state = watchedEvent.getState();
        Watcher.Event.EventType type = watchedEvent.getType();

        if (state == Watcher.Event.KeeperState.SyncConnected) {
            latch.countDown();
        }
    });
    latch.await();
    // 打印zk状态，如果不加 CountDownLatch，则会打印 connecting，因为是异步连接
    ZooKeeper.States state = zk.getState();
    switch (state) {
        case CONNECTING:
            System.out.println("connecting");
            break;
        case CONNECTED:
            System.out.println("connected");
            break;
        default:
            break;
    }
    // 创建节点
    zk.create("/test", "myData".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);

    // 同步获取数据
    byte[] node = zk.getData("/test", event -> System.out.println(event.toString()), new Stat());
    System.out.println(new String(node));

    // 设置数据
    zk.setData("/test", "newData".getBytes(), 0);

    // 异步获取数据
    zk.getData("/test", false, (rc, path, ctx, data, stat) ->
            System.out.println(new String(data)), "aa");

    // 阻塞线程 以便获取异步的数据
    Thread.sleep(100000);
}
```

## Zookeeper 分布式锁

redis 作为分布式锁的缺点：单点会挂，需要开启内存持久化，效率会变慢

zookeeper 分布式锁流程

1. 线程会在 zk 中创建临时有序节点
2. 节点序号最小的可以获得锁
3. 每个节点可以看到所有前面的节点，一旦锁被释放，zk只给第二个节点发事件回调

简单来说，每次都创建节点的ID会递增，ID越小越先被执行

代码实现

```java
@Data
public class ZkWatchLock implements Watcher, AsyncCallback.StringCallback,
        AsyncCallback.Children2Callback, AsyncCallback.StatCallback {

    private static ZooKeeper zk = ZkUtils.getInstance();

    private CountDownLatch latch = new CountDownLatch(1);

    private String pathName;

    private String threadName;

    public void lock() {
        // 创建lock有序节点，值是线程名
        zk.create("/lock", threadName.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL, this, "a");
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void unLock() {
        try {
            zk.delete(pathName, -1);
        } catch (InterruptedException | KeeperException e) {
            e.printStackTrace();
        }
        System.out.println(threadName + " unlock");
    }

    /**
     * Watcher
     */
    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == Event.EventType.NodeDeleted) {
            zk.getChildren("/", false, this, "w");
        }
    }

    /**
     * Children2Callback
     */
    @Override
    public void processResult(int rc, String path, Object ctx, List<String> children, Stat stat) {
        // 每一个节点都能看到自己前面的节点
        System.out.println(threadName + " look locks");
        for (String child : children) {
            System.out.print(child + " ");
        }
        System.out.println();

        // 将节点排序
        Collections.sort(children);

        // 判断是否是第一个节点
        int i = children.indexOf(pathName.substring(1));
        if (i == 0) {
            try {
                zk.setData("/", threadName.getBytes(), -1);
                latch.countDown();
            } catch (KeeperException | InterruptedException e) {
                e.printStackTrace();
            }
        } else {
            zk.exists("/" + children.get(i - 1), this, this, "c");
        }
    }

    /**
     * StringCallback
     */
    @Override
    public void processResult(int rc, String path, Object ctx, String name) {
        if (name != null) {
            System.out.println(threadName + "\tcreate node : " + name);
            pathName = name;
            zk.getChildren("/", false, this, "s");
        }
    }

    /**
     * StatCallback
     */
    @Override
    public void processResult(int rc, String path, Object ctx, Stat stat) {

    }
}
```

测试方法

```java
public class ZkLockClient {
    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                String threadName = Thread.currentThread().getName();
                ZkWatchLock zkWatchLock = new ZkWatchLock();
                zkWatchLock.setThreadName(threadName);
                //抢锁
                zkWatchLock.lock();
                //干活
                System.out.println(threadName + " working...");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //释放锁
                zkWatchLock.unLock();
            }).start();
        }
        Thread.sleep(15000);
    }
}
```

## 附录

Zookeeper 有个 sync 同步（可选项），客户端发起查询请求的时候，Zk会发起一个同步请求给 Leader，完成同步之后再返回，同步是基于回调的方式，不会阻塞。

zookeeper缺陷：Leader 宕机后 Follower 节点会重新进行选举，选举期间整个集群都是不可用的，这就导致了在选举期间注册服务瘫痪。

临时节点：和 Session 绑定，且 Session 也需要消耗事务ID，当连接上 zk 的时候会生成一个 Session，连接断开后 Session 消失

zk是有 Session 概念的，所以没有连接池的概念