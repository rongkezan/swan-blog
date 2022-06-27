---
title: Redis集群
date: {{ date }}
categories:
- 数据源
- Redis
---

## redis单机问题与解决

1. 单点故障：增加从机，只可以读不可以写
2. 容量有限：通过业务将redis拆分成多个
3. 压力较大：在同一种业务下再进行细分，如将每1kw的用户存入一个redis

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210213111000825.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 集群之间的通信
各个数据库相互通信，保存各个库中槽的编号数据

- 一次命中，直接返回

- 一次未命中，告知具体位置 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200203150519604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 集群配置

准备三个redis服务，6379 6380 6381

### 命令启动集群

```shell
# 主机正常启动
redis-server ./6379.conf
# 从机追随主机启动
redis-server ./6380.conf --replicaof 127.0.0.1 6379
redis-server ./6381.conf --replicaof 127.0.0.1 6379
# 如果没有哨兵模式，主机6379挂了，人工将从机6380切换为主机，并让6381追随它
127.0.0.1:6380> replicaof no one
127.0.0.1:6381> replicaof 127.0.0.1 6380
```

### 配置文件启动集群

1. 修改redis.conf
```shell
# 开启集群
cluster-enabled yes
# 设置集群配置文件，每个服务器要不一样
cluster-config-file node-6379.conf
# 设置下线时间
cluster-node-timeoout 10000
```
2. 依次启动集群服务
```shell
redis-server redis-6379.conf
```
3. 将redis服务连接起来
需要执行src目录下的redis-trib.rb，且需要ruby环境
下列命名表示1个master有1个slave，且一共有6个服务器
```shell
./redis-trib.rb create --replicas 1 \
127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \
127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384
```
## 哨兵模式

### 哨兵简介

哨兵（sentinel）是一个分布式系统，用于对主从结构中的每台服务器进行监控，当出现故障时通过投票机制选择新的master并将所有slave连接到新的master。

**作用**

1. 监控：不断的检查master和slave是否正常运行，master存活检测、master与slave运行情况检测
2. 通知：当被监控的服务器出现故障时，向其他客户端发送通知
3. 自动故障转移：断开master和slave连接，选取一个slave作master，将其他slave连接到新master，并告知客户端新的服务器地址

**说明**

- 哨兵也是一台redis服务器，只是不提供数据服务
- 通常哨兵的配置数量为单数

### 配置文件

26379.conf 26380.conf 26381.conf

```shell
# 指定端口 26379 26380 26381
port 26379
# 2个哨兵认为这个master挂了就挂了
sentinel monitor mymaster 127.0.0.1 6379 2
# 30秒未响应判断死亡
sentinel down-after-milliseconds mymaster 30000
```

启动哨兵服务器

```shell
redis-server ./26379.conf --sentinel
```

查看哨兵的通信，在master节点输入命令

```shell
PSUBSCRIBE *
```

可以看到以下输出，哨兵在询问master是否存活

```shell
3) "__sentinel__:hello"
4) "127.0.0.1,26379,36f7e5a15c4bda4ea9394a8b90c5c2c307123b25,0,mymaster,127.0.0.1,6379,0"
1) "pmessage"
```

