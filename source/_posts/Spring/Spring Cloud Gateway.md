---
title: Spring Cloud Gateway
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

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

