---
title: SpringCloud 链路追踪
date: {{ date }}
categories:
- 编程语言
- Java
---

如果能跟踪每个请求，中间请求经过哪些微服务，请求耗时，网络延迟，业务逻辑耗时等。我们就能更好地分析系统瓶颈、解决系统问题。

链路追踪目的：解决错综复杂的服务调用中链路的查看。排查慢服务。



市面上链路追踪产品大部分都是基于google的Dapper论文

zipkin,twitter：开源的，是严格按照谷歌的Dapper论文来的

pinpoint：韩国的 Naver 公司的

Cat：美团点评

EagleEye：淘宝

## 链路追踪要考虑的几个问题

1. 探针的性能消耗，尽量不影响服务本身。
2. 易用，开发可以很快接入，别浪费太多精力。
3. 数据分析，要实时分析，维度足够。

## Spring Cloud Sleuth

> Sleuth 是 Spring Cloud 的分布式跟踪解决方案

### 基本概念

1. span：基本工作单元，一次链路调用，创建一个span

   span 用一个64位 ID 唯一标识。包括：ID，描述，时间戳事件，spanId，span父id

   span 被启动和停止时，记录了时间信息，初始化span叫：root span，它的 span id 和 trace id 相等

2. trace：一组共享 `root span` 的 span 组成的树状结构称为 trace

   trace 也有一个64位 ID，trace 中所有 span 共享一个 trace id，类似于一颗 span 树。

3. annotation：用来记录事件的存在，核心annotation用来定义请求的开始和结束。

   1. CS：Client Send，客户端发起请求，客户端发起请求描述了span开始。

   2. SR：Server Received，服务端接到请求，服务端获得请求并准备处理它，SR-CS=网络延迟。

   3. SS：Server Send，服务器端处理完成，并将结果发送给客户端，表示服务器完成请求处理，响应客户端时，SS-SR=服务器处理请求的时间。

   4. CR：Client Received，客户端接受服务端信息，span结束的标识，客户端接收到服务器的响应，CR-CS=客户端发出请求到服务器响应的总时间。

其实数据结构是一颗树，从root span 开始。

### Sleuth 使用

pom

```xml
<!-- 引入sleuth依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

请求日志

```
--- 发起调用方 ---
[api-driver,1a409c98e7a3cdbf,1a409c98e7a3cdbf,true] 

--- 接收调用方 ---
[service-sms,1a409c98e7a3cdbf,b3d93470b5cf8434,true]

--- 参数解释 ---
[服务名称，traceId（一条请求调用链中 唯一ID），spanID（基本的工作单元，获取数据等），是否让zipkin收集和展示此信息]
```

## Spring Cloud Zipkin

### 基本概念

zipkin是twitter开源的分布式跟踪系统。

原理收集系统的时序数据，从而追踪微服务架构中系统延时等问题，还有一个友好的界面。

ZipKin的组成：Collector、Storage、Restful Api、Web UI

Sleuth 收集跟踪信息通过 http 请求发送给 Zipkin Server，Zipkin 将跟踪信息存储，以及提供RESTful API接口，Zipkin UI 通过调用 Api进行数据展示。默认内存存储，可以用mysql，ES等存储。

### ZipKin 使用

pom

```xml
<!-- zipkin -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

配置

```yaml
spring:
  zipkin:
    base-url: http://localhost:9411/
  sleuth:
    sampler:
      rate: 1  #采样比例1
```

下载启动 ZipKin

https://zipkin.io/pages/quickstart.html

```shell
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

访问地址：http://localhost:9411/

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210208165821768.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)