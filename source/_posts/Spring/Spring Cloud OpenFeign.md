---
title: Spring Cloud OpenFeign
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

## Feign

> 声明式、模板化的HTTP请求客户端

主要构建微服务消费端。只要使用OpenFeign提供的注解修饰定义网络请求的接口类，就可以使用该接口的实例发送 Restful 的网络请求。还可以集成Ribbon和Hystrix，提供负载均衡和断路器。

**实现**

pom

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>        
    <groupId>org.springframework.cloud</groupId>            
    <artifactId>spring-cloud-starter-openfeign</artifactId>            
</dependency>
```

服务端暴露接口到 `sardine-user-api`

```java
@FeignClient("sardine-user")
public interface UserClient {
    @PostMapping("identify")
    CommonResult<UserDto> identify(@RequestParam String token);
}
```

```java
@PostMapping("identify")
public CommonResult<UserDto> identify(@RequestParam(value = "token", required = false) String token) {
    UserDto userInfo = new UserInfo("1", "张三");
    return CommonResult.success(userInfo);
}
```

客户端引入 api 依赖

```xml
<!-- user api -->
<dependency>
    <groupId>com.sardine</groupId>
    <artifactId>sardine-user-api</artifactId>
    <version>1.0.0</version>
</dependency>
```

启动类加上注解

```java
@EnableFeignClients
```

客户端使用 UserClient

```java
@Autowired
private UserClient userClient;

@GetMapping("call")
public CommonResult<UserDto> call(){
    return userClient.identify(token).getData();
}
```

## 