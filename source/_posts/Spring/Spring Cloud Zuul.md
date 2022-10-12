---
title: Spring Cloud Zuul
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

## Zuul

基于 Servlet

### Zuul 概述

Zuul是Netflix开源的微服务网关，核心是一系列过滤器。这些过滤器可以完成以下功能。

1. 是所有微服务入口，进行分发。
2. 身份认证与安全：识别合法的请求，拦截不合法的请求。
3. 监控：在入口处监控，更全面。
4. 动态路由：动态将请求分发到不同的后端集群。
5. 压力测试：可以逐渐增加对后端服务的流量，进行测试。
6. 负载均衡
7. 限流：比如我每秒只要1000次，1001次就不让访问了。 
8. 服务熔断

zuul是跑在Tomcat上的，性能较低，可以使用读写分离，写请求通过zuul来实现

### Zuul 配置

引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

配置文件 

```yaml
eureka:
  client:
    service-url:
      # 需要和其他服务注册到同一个注册中心
      defaultZone: http://127.0.0.1:5000/eureka
zuul:
  routes:
    sardine-user: /user/**
    sardine-cookbook: /cookbook/**
    sardine-file: /file/**
  prefix: /api
  add-host-header: true # 携带域名信息
  sensitive-headers:    # 忽略头信息
```

启动类增加注解

```java
@EnableZuulProxy
public class ZuulApplication {
```

### Zuul 过滤器

```java
public class LoginFilter extends ZuulFilter {

    @Autowired
    private UserClient userClient;
    
    /**
     * 过滤器的类型: pre route post error
     */
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    /**
     * 过滤器的执行顺序: 返回值越小，优先级越高
     */
    @Override
    public int filterOrder() {
        return 10;
    }

    /**
     * 是否执行run方法
     */
    @Override
    public boolean shouldFilter() {
        RequestContext context = RequestContext.getCurrentContext();
        // 如果执行了context.setSendZuulResponse(false)方法，就不执行过滤器
        // 相当于不执行后面的过滤器了
        if (context != null && !context.sendZuulResponse())
            return false;
        return true;
    }

    /**
     * 过滤器的业务逻辑
     */
    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();
        String token = request.getHeader(authProperties.getTokenName());
        if (StringUtils.isBlank(token))
            return noAuthorization(context);
        UserDto user = userClient.identify(token).getRecord();
        if (user == null)
            return noAuthorization(context);
        return null;
    }

    /**
     * 拦截此次请求
     */
    private Object noAuthorization(RequestContext context){
        context.setSendZuulResponse(false);
        context.setResponseStatusCode(HttpStatus.SC_UNAUTHORIZED);
        context.setResponseBody("No Authorization");
        return null;
    }
}
```

## 