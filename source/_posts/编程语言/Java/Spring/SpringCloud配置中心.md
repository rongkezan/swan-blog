---
title: SpringCloud 配置中心
date: {{ date }}
categories:
- 编程语言
- Java
---

# Spring Cloud 配置中心

## Spring Cloud Config Center

1. 新建一个Git配置仓库，配置文件命名规则如下

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

2. 基本依赖和配置 pom

```xml
<!-- Spring Cloud Eureka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Spring Boot Actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-actuator</artifactId>
</dependency>
```

application.properties

```properties
server.port=5100
spring.application.name=config-center
spring.cloud.config.server.git.uri=https://<your-repository>.git
spring.cloud.config.label=master
eureka.client.service-url.defaultZone=http://eureka1.com:5000/eureka
```

3. 启动类增加注解

```java
@EnableConfigServer
```

4. 访问以下地址可以得到具体配置

```
{ 配置中心服务地址 }/master/file-dev.yml 
```

5. 客户端配置 pom

```xml
<!-- Spring Cloud Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

配置文件 `bootstrap.properties`

读取配置中心的master分支的配置文件 `sardine-file-dev.yml`

```properties
spring.application.name=sardine-file
spring.cloud.config.uri=http://127.0.0.1:5100/
spring.cloud.config.profile=dev
spring.cloud.config.label=master
```

6. 热更新

   6.1 手动配置热更新

   1. 开启 actuator 中的 refresh 端点
   2. Controller 中添加 @RefreshScope 注解
   3. 向客户端 `http://localhost:5005/actuator/refresh` 发送 Post 请求

   6.2 自动热更新

```xml
<!-- Spring Cloud Bus Amqp -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
```

向配置中心发送 Post 请求

`http://localhost:5100/actuator/bus-refresh` 

## Consul

## Apollo

## Nacos



