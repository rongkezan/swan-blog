---
title: Redis主从哨兵集群
date: {{ date }}
categories:
- Redis
---

## 主从模式

### 简介

简介：主从复制即将master中的数据及时有效地复制到slave中

特征：一个master可以有多个slave，一个slave只对应一个master。

职责：master写数据，同步数据到salve，slave读数据

### 示意图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200203094442305.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 工作流程

1. 建立连接

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200203131319126.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

2. 同步数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200203131306330.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)



### 主从配置

#### 配置方式

方式1：客户端发送命令

```sh
slaveof <master-ip> <master-port>
```

方式2：启动时添加参数

```sh
redis-server -slaveof <master-ip> <master-port>
```

方式3：修改配置文件`redis.conf`

```sh
slaveof <master-ip> <master-port>
```

#### 相关命令

```sh
# 查看所有信息
info

# 查看主从信息
info replication 

# 断开连接
slaveof no one
```

### 注意事项

1. 复制缓冲区大小设置不合理会导致数据溢出使主从数据不一致，主从数据不一致会导致全量复制，所以必须将复制缓冲区设置一个合理的大小。

```sh
# 复制缓冲区默认为1mb
repl-blocking-size 1mb
```

2. master单机内存占用建议使用50%-70%，留下30%-50%用于执行bgsave命令和创建缓冲区。
3. slave最好只对外开放读功能，不开放写功能

```sh
slave-serve-stale-data yes|no
```

## 哨兵模式

### 简介

哨兵（sentinel）是一个分布式系统，用于对主从结构中的每台服务器进行监控，当出现故障时通过投票机制选择新的master并将所有slave连接到新的master。哨兵也是一台redis服务器，只是不提供数据服务，通常哨兵的配置数量为单数。

1. 监控：不断的检查master和slave是否正常运行，master存活检测、master与slave运行情况检测
2. 通知：当被监控的服务器出现故障时，向其他客户端发送通知
3. 自动故障转移：断开master和slave连接，选取一个slave作master，将其他slave连接到新master，并告知客户端新的服务器地址

### 配置文件

```shell
# 指定端口 26379
port 26379
# 监控主节点，名称为mymaster，至少需要两个哨兵同意才会进行故障转移
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

### 整合 Spring Boot

1. pom依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

2. 配置文件

```yaml
server:
  port: 8000
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - 127.0.0.1:26379
        - 127.0.0.1:26380
        - 127.0.0.1:26381
```

## 集群模式

### 示意图

各个数据库相互通信，保存各个库中槽的编号数据

- 一次命中，直接返回

- 一次未命中，告知具体位置 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200203150519604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 集群配置

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
   
   > 需要执行src目录下的redis-trib.rb，且需要ruby环境
   > 下列命名表示1个master有1个slave，且一共有6个服务器

```shell
./redis-trib.rb create --replicas 1 \
127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \
127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384
```

