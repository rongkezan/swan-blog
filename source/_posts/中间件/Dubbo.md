---
title: Dubbo
date: {{ date }}
categories:
- 中间件
- Zookeeper
---

## Dubbo各层说明

第一层：service层，接口层，给服务提供者和消费者来实现的

第二层：config层，配置层，主要是对dubbo进行各种配置的

第三层：proxy层，服务代理层，透明生成客户端的stub和服务单的skeleton

第四层：registry层，服务注册层，负责服务的注册与发现

第五层：cluster层，集群层，封装多个服务提供者的路由以及负载均衡，将多个实例组合成一个服务

第六层：monitor层，监控层，对rpc接口的调用次数和调用时间进行监控

第七层：protocol层，远程调用层，封装rpc调用

第八层：exchange层，信息交换层，封装请求响应模式，同步转异步

第九层：transport层，网络传输层，抽象mina和netty为统一接口

第十层：serialize层，数据序列化层

## Dubbo 执行流程

服务的提供方会向注册中心注册自己提供的服务，当消费者在启动时，就会向注册中心订阅自己所需要的服务，如果服务提供方有数据变更等，注册中心将基于长连接的形式推送变更数据给消费者。

## Dubbo 同步调用

**同步调用流程**

1. Consumer 业务线程调用远程接口，向 Provider 发送请求，同时当前线程处于`阻塞`状态；
2. Provider 接到 Consumer 的请求后，开始处理请求，将结果返回给 Consumer；
3. Consumer 收到结果后，当前线程继续往后执行。

**Dubbo底层是如何阻塞的**

> Dubbo 的底层 IO 操作都是异步的

Consumer 端发起调用后，得到一个 Future 对象。对于同步调用，业务线程通过`Future.get(timeout)`，阻塞等待 Provider 端将结果返回。`timeout`则是 Consumer 端定义的超时时间。当结果返回后，会设置到此 Future，并唤醒阻塞的业务线程；当超时时间到结果还未返回时，业务线程将会异常返回。

## Dubbo 负载均衡的实现

FailoverCluster类的invoke调用，对invocation进行了拦截实现去实现负载均衡

## 注册中心挂了可以继续通信吗

可以通信的，启动dubbo时，消费者会从zk拉取注册的生产者的地址接口等数据，缓存在本地。每次调用时，按照本地存储的地址进行调用。

## Dubbo 心跳机制

目的：维持provider和consumer之间的长连接

实现：dubbo心跳时间heartbeat默认是1s，超过heartbeat时间没有收到消息，就发送心跳消 息(provider，consumer一样),如果连着3次(heartbeatTimeout为heartbeat*3)没有收到心跳响应，provider会关闭channel，而consumer会进行重连;不论是provider还是consumer的心跳检测都是通过启动定时任务的方式实现；

Dubbo的zookeeper做注册中心，如果注册中心全部挂掉，发布者和订阅者还能通信吗？

注册中心对等集群，任意一台宕机后，将会切换到另一台；注册中心全部宕机后，服务的提供者和消费者仍能通过本地缓存通讯。服务提供者无状态，任一台 宕机后，不影响使用；服务提供者全部宕机，服务消费者会无法使用，并无限次重连等待服务者恢复；
挂掉是不要紧的，但前提是你没有增加新的服务，如果你要调用新的服务，则是不能办到的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210128173522387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## RPC协议

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106221257207.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

| 协议名称   | 实现描述                                     | 连接                                     | 使用场景                                                     |
| ---------- | -------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| dubbo      | netty                                        | 单一长连接和NIO异步传输                  | 小数据量大并发的服务调用，消费者比提供者多，不适合传送大数据量的服务，比如文件、传视频 |
| rmi        | 采用JRM 作为通讯协议                         | 多连接，短连接，TCP/IP，BIO              | 可传文件，不支持防火墙穿透                                   |
| hessian    | hessian二进制序列化                          | 多连接，短连接，传输协议：HTTP，同步传输 | 提供者比消费者多 ，可传文件，跨语言传输                      |
| http       | 表单序列化                                   | 多连接，短连接，HTTP，同步传输           | 提供者大于消费者，数据包混合                                 |
| webservice | SOAP文件序列化                               | 多连接，短连接，HTTP，同步传输           | 系统集成，跨语言调用                                         |
| thrift     | 与thrift RPC实现集成，并在基础上修改了报文头 | 长连接、NIO异步传输                      |                                                              |

