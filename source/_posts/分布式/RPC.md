---
title: RPC
date: {{ date }}
categories:
- 分布式
---

## RPC

### RPC调用流程

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/d409f534088344a7be194861c766ab73.png)

### RPC框架

Dubbo、gRPC、Thrift、HSF (High Speed Service Framework)

## Dubbo

### Dubbo 协议

| 协议名称   | 实现描述                                     | 连接                                     | 使用场景                                                     |
| ---------- | -------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| dubbo      | netty                                        | 单一长连接和NIO异步传输                  | 小数据量大并发的服务调用，消费者比提供者多，不适合传送大数据量的服务，比如文件、传视频 |
| rmi        | 采用 JRM 作为通讯协议                        | 多连接，短连接，TCP/IP，BIO              | 可传文件，不支持防火墙穿透                                   |
| hessian    | hessian二进制序列化                          | 多连接，短连接，传输协议：HTTP，同步传输 | 提供者比消费者多 ，可传文件，跨语言传输                      |
| http       | 表单序列化                                   | 多连接，短连接，HTTP，同步传输           | 提供者大于消费者，数据包混合                                 |
| webservice | SOAP文件序列化                               | 多连接，短连接，HTTP，同步传输           | 系统集成，跨语言调用                                         |
| thrift     | 与thrift RPC实现集成，并在基础上修改了报文头 | 长连接、NIO异步传输                      |                                                              |

### Dubbo 服务发现

<img src="https://dubbo.apache.org/imgs/architecture.png" alt="//imgs/architecture.png" style="zoom:50%;" />

1. 服务提供者启动初始化：注册中心注册服务
2. 服务消费者启动初始化：向注册中心订阅服务，缓存到本地，所以即使注册中心挂了也可以进行调用
3. 当服务提供者有变更，注册中心会基于长链接的方式通知消费者
4. 服务消费者会基于拉取到的服务调用服务提供者
5. 每次调用的信息会定时地发送到监控中心

### Dubbo 同步调用

**同步调用流程**

1. Consumer 业务线程调用远程接口，向 Provider 发送请求，同时当前线程处于`阻塞`状态；
2. Provider 接到 Consumer 的请求后，开始处理请求，将结果返回给 Consumer；
3. Consumer 收到结果后，当前线程继续往后执行。

**Dubbo底层是如何阻塞的**

> Dubbo 的底层 IO 操作都是异步的

Consumer 端发起调用后，得到一个 Future 对象。对于同步调用，业务线程通过`Future.get(timeout)`，阻塞等待 Provider 端将结果返回。`timeout`则是 Consumer 端定义的超时时间。当结果返回后，会设置到此 Future，并唤醒阻塞的业务线程；当超时时间到结果还未返回时，业务线程将会异常返回。

### Dubbo 负载均衡的实现

FailoverCluster类的invoke调用，对invocation进行了拦截实现去实现负载均衡

### Dubbo 心跳机制

目的：维持provider和consumer之间的长连接

实现：dubbo心跳时间heartbeat默认是1s，超过heartbeat时间没有收到消息，就发送心跳消息(provider，consumer一样),如果连着3次(heartbeatTimeout为heartbeat*3)没有收到心跳响应，provider会关闭channel，而consumer会进行重连;不论是provider还是consumer的心跳检测都是通过启动定时任务的方式实现；

Dubbo的zookeeper做注册中心，如果注册中心全部挂掉，发布者和订阅者还能通信吗？

注册中心对等集群，任意一台宕机后，将会切换到另一台；注册中心全部宕机后，服务的提供者和消费者仍能通过本地缓存通讯。服务提供者无状态，任一台 宕机后，不影响使用；服务提供者全部宕机，服务消费者会无法使用，并无限次重连等待服务者恢复；
挂掉是不要紧的，但前提是你没有增加新的服务，如果你要调用新的服务，则是不能办到的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210128173522387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)
