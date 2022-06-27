---
title: SpringCloud 服务网关
date: {{ date }}
categories:
- 编程语言
- Java
---

# Spring Cloud 服务网关

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

## Gateway

基于 Netty，Spring WebFlux 响应式编程，在请求的时候就封装好了Response

### 基本使用

Predicates 断言

> 多个断言可以配合使用

```yaml
predicates:
  - Path=/img/**	# 匹配路径
  - Query=foo,ba	# 匹配参数，可使用正则
  - Method=get		# 匹配请求方式
  - Host=demo.com	# 匹配Host
  - Cookie=aaa		# 匹配Cookie
```

pom

```xml
<!-- Spring Cloud Gateway -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

配置文件

```yaml
spring:
  application:
    name: sardine-gateway
  cloud:
    gateway:
      routes:
        - id: my-route
          uri: http://localhost:7012
          predicates:
            - Path=/gateway/**      # 将后缀是此路径的转发到uri
          filters:
            - StripPrefix=1         # 去掉1个前缀
```

### 整合 Eureka

**使用默认路由**

pom

```yaml
server:
  port: 5001
spring:
  application:
    name: sardine-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7000/eureka
```

默认是根据 Eureka 的服务名进行转发（注意要大写0）：http://localhost:5001/SARDINE-USER/hello

**使用自定义路由**

```yaml
server:
  port: 5001
spring:
  application:
    name: sardine-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: sardine-user
          predicates:
            - Path=/user/**
          uri: lb://SARDINE-USER
          filters:
            - StripPrefix=1
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7000/eureka
```

访问该地址可以得到同样的效果： http://localhost:5001/user/hello

### 自定义负载均衡策略

定义自定义负载均衡策略

```java
public class MyRule extends AbstractLoadBalancerRule {
    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {

    }

    @Override
    public Server choose(Object o) {
        System.out.println("MyRule");
        return null;
    }
}
```

在 gateway 服务中增加以下配置

```yaml
SARDINE-USER:
  ribbon:
    NFLoadBalancerRuleClassName: com.sardine.gateway.config.MyRule
```

### 自定义路由

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder locatorBuilder){
    return locatorBuilder.routes()
        .route(p -> p.path("/u/**").filters(f -> f.stripPrefix(1)).uri("lb://SARDINE-USER")).build();
}
```

### 过滤器

```java
@Component
public class MyFilter implements Ordered, GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        System.out.println("执行过滤器");
        ServerHttpRequest request = exchange.getRequest();
        MultiValueMap<String, String> queryParams = request.getQueryParams();
        List<String> list = queryParams.get("id");
        if (list == null || list.size() == 0){
            System.out.println("非法请求");
//            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
//            return exchange.getResponse().setComplete();
            DataBuffer dataBuffer = exchange.getResponse().bufferFactory().wrap("Illegal Request".getBytes());
            return exchange.getResponse().writeWith(Mono.just(dataBuffer));
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

### 限流

#### 整合Guava

限流实现类

```java
@Primary
@Component
public class DefaultRateLimiter extends AbstractRateLimiter<DefaultRateLimiter.Config> {

    private final RateLimiter limiter = RateLimiter.create(1);

    public DefaultRateLimiter() {
        super(Config.class, "default-rate-limiter", new ConfigurationService());
    }

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        Config config = getConfig().get(routeId);
        limiter.setRate(Objects.isNull(config.getPermitsPerSecond()) ? 1 : config.getPermitsPerSecond());
        boolean isAllow = limiter.tryAcquire();
        return Mono.just(new Response(isAllow, new HashMap<>()));
    }

    public static class Config {

        @DecimalMin("0.1")
        private Double permitsPerSecond;

        public Double getPermitsPerSecond() {
            return permitsPerSecond;
        }

        public Config setPermitsPerSecond(Double permitsPerSecond) {
            this.permitsPerSecond = permitsPerSecond;
            return this;
        }
    }
}
```

配置需要根据什么维度进行限流

```java
@Component
public class RateLimitConfig {
    @Bean
    public KeyResolver userKeyResolver(){
        // 获取请求用户IP作为限流key
        return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
    }
}
```

配置文件

```yaml
spring:
  application:
    name: sardine-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
      - id: sardine-user
        predicates:
        - Path=/user/**
        uri: lb://SARDINE-USER
        filters:
        - StripPrefix=1
        - name: RequestRateLimiter
          args:
          	# 限流的实现类，Spring的EL表达式，拿到这个Bean
            rate-limiter: '#{@defaultRateLimiter}'
            # 需要限流的规则
            key-resolver: '#{@userKeyResolver}'
            # 每秒发放令牌个数
            default-rate-limiter.premitPerSecond: 0.5
```

#### 使用Filter 实现令牌桶算法

1. 修改pom 添加redis依赖
2. 添加reids key 的解析器即key-resolver 解析类
3. 调整配置文件

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```java
@Configuration
public class KeyResolverConfiguration {
 
    @Bean
    public KeyResolver pathKeyResolver(){
        return new KeyResolver() {
            @Override
            public Mono<String> resolve(ServerWebExchange exchange) {
                return Mono.just(exchange.getRequest().getPath().toString());
            }
        };
    }
}
```

```properties
spring:
  application:
    name: api-gateway-server #服务名称
  redis:
    host: 127.0.0.1
    pool: 6379
    database: 0
  cloud:
    gateway:
      routes:
      #配置路由： 路由id,路由到微服务的uri,断言（判断条件）
        - id: product-service #保持唯一
          #uri: http://127.0.0.1:8001 #目标为服务地址
          uri: lb://cloud-payment-service # lb:// 根据服务名称从注册中心获取请求地址路径
          predicates:
            #- Path=/payment/** #路由条件 path 路由匹配条件
            - Path=/product-service/** #给服务名称前加上一个固定的应用分类路径 将该路径转发到 http://127.0.0.1:8001/payment/get/1
          filters: #配置路由过滤器  http://127.0.0.1:8080/product-service/payment/get/1 -> http://127.0.0.1:8001/payment/get/1
          - name: RequestRateLimiter
            args:
              #使用SpEL从容器中获取对象
              key-resolver: '#{@pathKeyResolver}'
              #桶令牌每秒产生平均速率
              redis-rate-limiter.replenishRate: 1
              #令牌桶的上限
              redis-rate-limiter.burstCapacity: 2
          - RewritePath=/product-service/(?<segment>.*),/$\{segment} #路径重写的过滤器，在yml中$写为 $\
```

