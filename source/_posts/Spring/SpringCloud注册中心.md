---
title: SpringCloud 注册中心
date: {{ date }}
categories:
- Spring
tags:
- SpringCloud
---

## 注册中心和微服务的关系

1. 注册：每个微服务启动时，将自己的网络地址等信息注册到注册中心，注册中心会存储（内存中）这些信息。
2. 获取服务注册表：服务消费者从注册中心，查询服务提供者的网络地址，并使用该地址调用服务提供者，为了避免每次都查注册表信息，所以client会定时去server拉取注册表信息到缓存到client本地。
3. 心跳：各个微服务与注册中心通过某种机制（心跳）通信，若注册中心长时间和服务间没有通信，就会注销该实例。
4. 调用：实际的服务调用，通过注册表，解析服务名和具体地址的对应关系，找到具体服务的地址，进行实际调用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021020610401060.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## Spring Cloud Eureka

### Eureka 概念

Eureka注册中心各个节点是平等的，节点挂掉不会影响剩余节点的正常工作，只要有一台Eureka还在，就能保证注册服务可用，只不过查询到的数据可能不会最新的。Eureka还有自我保护机制，如果在15分钟内超过85%的节点都没有正常心跳，那么Eureka就认为客户端与注册中心出现了故障，此时会出现以下几种情况：

1. Eureka不再从注册列表移除因为长时间没收到心跳而应该过期的服务
2. Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上（保证当前节点依然可用）
3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

**Eureka自我保护机制**

某时刻某一个微服务不可用了，eureka不会立即清理，依旧会对该微服务的信息进行保存

默认EurekaServer在一定时间内没有接收到某个微服务实例的心跳，EurekaServer将会注销该实例（默认90秒）

当网络分区故障发生时，微服务与EurekaServer之间无法正常通信，但微服务本身其实是健康的。当EurekaServer节点在短时间内丢失过多客户端时，那么这个节点就会进入自我保护模式，EurekaServer会保护服务注册表中的信息，不再注销任何微服务。当它收到的心跳数重新恢复到阈值以上时，该Eureka Server节点就会自动退出自我保护模式。

在Spring Cloud中，可以使用eureka.server.enable-self-preservation = false 禁用自我保护模式。

服务少的话要关闭自我保护，服务多的话要开启自我保护，原因是服务多的话如果发现服务挂了其实是不一定真的挂了，可以是由于网络波动等因素没调到服务，所以需要触发自我保护机制。

**Eureka 三级缓存**

| 缓存                  | 类型                     | 说明                                                         |
| --------------------- | ------------------------ | ------------------------------------------------------------ |
| **registry**          | ConcurrentHashMap        | **实时更新**，类AbstractInstanceRegistry成员变量，UI端请求的是这里的服务注册信息 |
| **readWriteCacheMap** | Guava Cache/LoadingCache | **实时更新**，类ResponseCacheImpl成员变量，缓存时间180秒     |
| **readOnlyCacheMap**  | ConcurrentHashMap        | **周期更新**，类ResponseCacheImpl成员变量，默认每**30s**从readWriteCacheMap更新，Eureka client默认从这里更新服务注册信息，可配置直接从readWriteCacheMap更新 |

开启 readOnlyCacheMap 是为了保证高可用，如果有大量请求进来会先往 readWriteCacheMap 里加，而不会影响 readOnlyCacheMap 的读

### Eureka Restful 服务调用

官方文档

https://github.com/Netflix/eureka/wiki/Eureka-REST-operations

| **Operation**                                                | **HTTP action**                                              | **Description**                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Register new application instance                            | POST /eureka/v2/apps/**appID**                               | Input: JSON/XMLpayload HTTPCode: 204 on success              |
| De-register application instance                             | DELETE /eureka/v2/apps/**appID**/**instanceID**              | HTTP Code: 200 on success                                    |
| Send application instance heartbeat                          | PUT /eureka/v2/apps/**appID**/**instanceID**                 | HTTP Code: * 200 on success * 404 if **instanceID**doesn’t exist |
| Query for all instances                                      | GET /eureka/v2/apps                                          | HTTP Code: 200 on success Output: JSON/XML                   |
| Query for all **appID** instances                            | GET /eureka/v2/apps/**appID**                                | HTTP Code: 200 on success Output: JSON/XML                   |
| Query for a specific **appID**/**instanceID**                | GET /eureka/v2/apps/**appID**/**instanceID**                 | HTTP Code: 200 on success Output: JSON/XML                   |
| Query for a specific **instanceID**                          | GET /eureka/v2/instances/**instanceID**                      | HTTP Code: 200 on success Output: JSON/XML                   |
| Take instance out of service                                 | PUT /eureka/v2/apps/**appID**/**instanceID**/status?value=OUT_OF_SERVICE | HTTP Code: * 200 on success * 500 on failure                 |
| Move instance back into service (remove override)            | DELETE /eureka/v2/apps/**appID**/**instanceID**/status?value=UP (The value=UP is optional, it is used as a suggestion for the fallback status due to removal of the override) | HTTP Code: * 200 on success * 500 on failure                 |
| Update metadata                                              | PUT /eureka/v2/apps/**appID**/**instanceID**/metadata?key=value | HTTP Code: * 200 on success * 500 on failure                 |
| Query for all instances under a particular **vip address**   | GET /eureka/v2/vips/**vipAddress**                           | * HTTP Code: 200 on success Output: JSON/XML  * 404 if the **vipAddress**does not exist. |
| Query for all instances under a particular **secure vip address** | GET /eureka/v2/svips/**svipAddress**                         | * HTTP Code: 200 on success Output: JSON/XML  * 404 if the **svipAddress**does not exist. |

### Eureka 单节点服务端配置说明

```yaml
eureka: 
  instance:
    # 表示将自己的ip注册到EurekaServer上。不配置或false，表示将操作系统的hostname注册到server
    prefer-ip-address: true
  client:
    # 是否将自己注册到Eureka Server,默认为true，由于当前就是server，故而设置成false，表明该服务不会向eureka注册自己的信息
    register-with-eureka: false
    # 是否从eureka server获取注册信息，由于单节点，不需要同步其他节点数据，用false
    fetch-registry: false
    # 设置服务注册中心的URL，用于client和server端交流
    service-url:                      
      defaultZone: http://root:root@eureka-7901:7901/eureka/
  server:
  	# 关闭自我保护
    enable-self-preservation: false
    # 自我保护阈值
    renewal-percent-threshold: 0.8
    # 剔除服务的时间间隔
    viction-interval-timer-in-ms: 1000
    # 关闭缓存
    use-read-only-response-cache: false
```

### Eureka Client 注册

pom.xml

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

application.yml

```yaml
#注册中心
eureka: 
  client:
    #设置服务注册中心的URL
    service-url:                      
      defaultZone: http://root:root@localhost:7900/eureka/
```

不想注册，设置成false即可

```yaml
spring: 
  cloud:
    service-registry:
      auto-registration:
        enabled: false
```

注册成功：

```
DiscoveryClient_API-LISTEN-ORDER/api-listen-order:30.136.133.9:port - registration status: 204
```

### Eureka 集群的本地搭建

引入依赖 eureka server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

修改本地hosts文件

```
127.0.0.1 eureka1.com
127.0.0.1 eureka2.com
```

application.yml

```yaml
spring:
  profiles:
    active: 01
---
server:
  port: 5001
spring:
  application:
    name: eureka-demo
  profiles: 01
eureka:
  instance:
    hostname: eureka1.com
  client:
    service-url:
      defaultZone: http://eureka2.com:5002/eureka
---
server:
  port: 5002
spring:
  application:
    name: eureka-demo
  profiles: 02
eureka:
  instance:
    hostname: eureka2.com
  client:
    service-url:
      defaultZone: http://eureka1.com:5001/eureka
```

启动类

```java
@SpringBootApplication
@EnableEurekaServer
public class RegistryApplication {
    public static void main(String[] args) {
        SpringApplication.run(RegistryApplication.class, args);
    }
}
```

**多节点 Eureka 注意事项**

> peer：A 同步信息给 B，A 是 B 的peer

举例说明：

```yaml
eureka:
  instance:
    hostname: eureka2.com
  client:
    service-url:
      defaultZone: http://eureka1.com:5001/eureka # 这个地址就是该服务的 peer
```

有3个服务 A B C，注册方式 A -> B，B->C，C->A，结果注册信息并不同步

启动流程：

1. 拉取 peer 的注册表，A 启动会拉取 B 的注册表
2. 把自己注册到 peer 上，A 会把自己注册到 B 的注册表上
3. peer 会把自己的注册表同步给 peer 的 peer，B 会把 A 同步给 C

### Eureka 架构图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206111125938.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### Eureka 客户端工作流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206141833488.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### Eureka 原理

> Eureka 存储了每个客户端的注册信息
>
> EurekaClient 从 EurekaServer 同步获取服务注册列表，通过一定的规则选择一个服务进行调用

#### 名词解释

**服务提供者**：是一个eureka client，向Eureka Server注册和更新自己的信息，同时能从Eureka Server注册表中获取到其他服务的信息。

**服务注册中心**：提供服务注册和发现的功能。每个 Eureka Client 向 Eureka Server 注册自己的信息，也可以通过Eureka Server获取到其他服务的信息达到发现和调用其他服务的目的。

**服务消费者**：是一个 Eureka Client，通过 Eureka Server 获取注册到其上其他服务的信息，从而根据信息找到所需的服务发起远程调用。

**同步复制**：Eureka Server 之间注册表信息的同步复制，使 Eureka Server 集群中不同注册表中服务实例信息保持一致。

**远程调用**：服务客户端之间的远程调用。

**注册**：Client 向 Server 注册自身的元数据以供服务发现。

**续约**：Eureka客户端需要每30秒发送一次心跳来续租，通过发送心跳到 Server 以维持和更新注册表中服务实例元数据的有效性。当在一定时长内，Server 没有收到 Client 的心跳信息，将默认服务下线，会把服务实例的信息从注册表中删除。

**下线**：Client 在关闭时主动向 Server 注销服务实例元数据，这时 Client 的服务实例数据将从 Server 的注册表中删除。

**获取注册表**：Client 向 Server 请求注册表信息，用于服务发现，从而发起服务间远程调用。

**元数据**：Eureka的元数据有两种：标准元数据和自定义元数据。

标准元数据：主机名、IP地址、端口号、状态页和健康检查等信息，这些信息都会被发布在服务注册表中，用于服务之间的调用。

自定义元数据：可以使用eureka.instance.metadata-map配置，这些元数据可以在远程客户端中访问，但是一般不改变客户端行为，	除非客户端知道该元数据的含义。

可以在配置文件中对当前服务设置自定义元数据，可后期用户个性化使用

元数据可以配置在 eureka 服务器和 eureka 客户端上

#### 如何自己实现一个注册中心

- 客户端：拉取注册表，从注册表里选一个调用

- 服务端：

  - 定义注册表：`Map<name, Map<id, InstanceInfo>>`
  - 别人可以向我注册自己的信息
  - 别人可以从我这里拉取他人的信息
  - 我和我的同类可以共享注册表

  eureka是用：jersey实现，也是个mvc框架。我们可以自己写个spring boot web实现。

### Eureka 源码

#### Eureka Server 启动

通过注解 @EnableEurekaServer 导入了 EurekaServerMarkerConfiguration 类

```java
@EnableEurekaServer -> @Import(EurekaServerMarkerConfiguration.class)
```

在 EurekaServerMarkerConfiguration 类中只是新建了一个 Marker

```java
@Bean
public Marker eurekaServerMarkerBean() {
    return new Marker();
}

class Marker {

}
```

在 Erueka Server 的 jar 包下的 spring.facoties 中发现导入了 EurekaServerAutoConfiguration 配置类，而该配置类生效的条件是 Marker 类的存在

```java
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
```

同时在 EurekaServerAutoConfiguration 中还导入了 EurekaServerInitializerConfiguration

```java
@Import(EurekaServerInitializerConfiguration.class)
```

而导入这个类后会直接执行其 start 方法，原因是它实现了 SmartLifecycle 接口

```java
public class EurekaServerInitializerConfiguration
		implements ServletContextAware, SmartLifecycle, Ordered {

	public void start() {
		// ...
	}
}
```

#### Eureka Server 注册实例

ApplicationResource#addInstance

```java
registry.register -> PeerAwareInstanceRegistryImpl#register ->
AbstractInstanceRegistry#register
```

在 AbstractInstanceRegistry 这个类中可以找到存放实例信息的map

```java
// map<服务名, map<实例ID，实例信息>>
ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> register
```

#### Eureka Server 拉取实例

ApplicationResource#getApplication

```
responseCache.get(cacheKey) -> getValue(key, useReadOnlyCache) ->
```

```java
// 如果开启了readOnly缓存
if (useReadOnlyCache) {
    // 先从readOnly缓存里取
    final Value currentPayload = readOnlyCacheMap.get(key);
    if (currentPayload != null) {
        payload = currentPayload;
    } else {
        // 取不到再从readWrite缓存里取
        payload = readWriteCacheMap.get(key);
        readOnlyCacheMap.put(key, payload);
    }
} else {
    // 如果没开启，则直接从readWrite缓存里取
    payload = readWriteCacheMap.get(key);
}
```
