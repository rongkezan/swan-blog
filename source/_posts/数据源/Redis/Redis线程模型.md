---
title: Redis线程模型
date: {{ date }}
categories:
- 数据源
- Redis
---

> NIO异步单线程

## 为什么redis单线程还能支撑高并发

1. IO多路复用程序只负责监听所有的socket产生的请求，有人发过来请求就直接放入队列中。
2. 事件处理器是基于纯内存操作的。
3. 单线程反而避免了多线程频繁切换上下文问题。

## 客户端与redis的通信的一次流程

1. 在redis初始化的时候，redis会将连接应答处理器跟AE_READABLE事件关联起来，接着如果一个客户端与redis发起连接，此时会产生一个AE_READABLE事件，然后由连接应答处理器来处理与客户端的连接，创建客户端对应的socket，同时将这个socket的AE_READABLE事件跟命令请求处理器关联起来。
2. 当客户端想redis发起请求的时候，首先会在socket产生一个AE_READABLE事件，然后由命令请求处理器来处理，这个命令请求处理器就会从socket中读取请求相关数据，然后进行执行和处理。
3. 接着redis这边准备好了给客户端的相应数据之后，就会将socket的AE_WRITEABLE事件跟命令回复处理器关联起来，当客户端这边准备好读取相应数据时，就会在socket上产生一个AE_WRITEABLE事件，会由命令回复处理器来处理，就是准备好写入socket，供客户端来读取。
4. 命令回复处理器写完之后，就会删除这个socket的AE_WRITEABLE事件和命令回复处理器的关联关系。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121916394029.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 附录

redis在线程有顺序性

**Redis是二进制安全的**

客户端通过socket访问redis，redis拿到字节流

