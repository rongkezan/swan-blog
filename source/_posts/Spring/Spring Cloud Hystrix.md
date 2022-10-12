---
title: SpringCloud Hystrix
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

## Hystrix

> 容错组件 实现了超时机制和断路器模式

### 1. 主要功能

1. 为系统提供保护机制。在依赖的服务出现高延迟或失败时，为系统提供保护和控制。
2. 防止雪崩。
3. 包裹请求：使用HystrixCommand（或HystrixObservableCommand）包裹对依赖的调用逻辑，每个命令在独立线程中运行。
4. 跳闸机制：当某服务失败率达到一定的阈值时，Hystrix可以自动跳闸，停止请求该服务一段时间。
5. 资源隔离：Hystrix为每个请求都的依赖都维护了一个小型线程池，如果该线程池已满，发往该依赖的请求就被立即拒绝，而不是排队等候，从而加速失败判定。防止级联失败。
6. 快速失败：Fail Fast。同时能快速恢复。侧重点是：（不去真正的请求服务，发生异常再返回），而是直接失败。
7. 监控：Hystrix可以实时监控运行指标和配置的变化，提供近实时的监控、报警、运维控制。
8. 回退机制：fallback，当请求失败、超时、被拒绝，或当断路器被打开时，执行回退逻辑。回退逻辑我们自定义，提供优雅的服务降级。
9. 自我修复：断路器打开一段时间后，会自动进入“半开”状态，可以进行打开，关闭，半开状态的转换。前面有介绍。

### 2. 独立使用

```java
public class HystrixTestService extends HystrixCommand<String> {

    protected HystrixTestService(HystrixCommandGroupKey group) {
        super(group);
    }

    @Override
    protected String run() throws Exception {
        System.out.println("执行逻辑");
        // 当执行 1/0 后抛出异常会执行 Fallback 逻辑，否则执行正常逻辑
        int i = 1 / 0;
        return "ok";
    }

    @Override
    protected String getFallback() {
        return "Fallback Function";
    }

    public static void main(String[] args) {
        Future<String> futureResult = new HystrixTestService(HystrixCommandGroupKey.Factory.asKey("ext")).queue();
        try {
            String result = futureResult.get();
            System.out.println("程序结果："+result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

### 3. 整合 RestTemplate

```java
// 启动类增加注解 @EnableCircuitBreaker
@EnableCircuitBreaker
public class UserApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }
}
```

```java
@HystrixCommand(fallbackMethod = "fallback")
public String alive() {
	// 自动处理URL
	RestTemplate restTemplate = new RestTemplate();
	String url ="http://user-provider/User/alive";
	String result = restTemplate.getForObject(url, String.class);
	return result;
}

public String fallback() {	
	return "Fallback Function";
}
```

### 4. 整合 Fegin

```properties
# 配置开启 Hystrix
feign.hystrix.enabled=true
```

```java
@FeignClient(name = "user-provider",fallback = AliveBack.class)
public interface ConsumerApi {

	@GetMapping(value = "/user/alive")
	public String alive();
	
	@GetMapping(value = "/user/getById")
	public String getById(Integer id);
}
```

```java
@Component
public class AliveBack implements ConsumerApi{

	@Override
	public String alive() {
		return "call exception";
	}

	@Override
	public String getById(Integer id) {
		return null;
	}
}
```

### 5. Hystrix Dashboard

```xml
<!-- Spring Boot Actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>        
<!-- Spring Cloud Hystrix Dashboard -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

```java
// 启动类增加注解
@EnableHystrixDashboard
```

```properties
# 增加配置
management.endpoints.web.exposure.include=*
hystrix.dashboard.proxy-stream-allow-list=*
```

访问 http://localhost:8001/actuator/hystrix.stream 查看请求信息

访问 http://localhost:8001/hystrix 输入上面的url可以可视化地观察请求信息

### 6. 隔离策略

隔离策略：信号量、线程池，默认使用线程池

**线程池隔离**

概念：有1000个商品服务信息的并发请求，但是商品服务线程池中只有10个线程，那么最多只会用这10个线程去处理，不会将Tomcat中的所有线程都耗尽。

适用场景：耗时长的请求

**信号量隔离**

概念：每次请求通过计数信号进行限制，当信号大于请求数时，调用Fallback接口快速返回。获取到信号的线程继续访问，访问完成后归还信号，信号获取失败直接Fallback。

适用场景：耗时短的请求

| 隔离方式   | 是否支持超时                                               | 是否支持熔断                                                 | 隔离原理             | 是否是异步调用                        | 资源消耗                                    |
| :--------- | :--------------------------------------------------------- | :----------------------------------------------------------- | :------------------- | :------------------------------------ | :------------------------------------------ |
| 线程池隔离 | 支持,可直接返回                                            | 支持,当线程池到达maxSize后,再请求会触发fallback接口进行熔断  | 每个服务单独用线程池 | 可以是异步,也可以是同步。看调用的方法 | 大,大量线程的上下文切换，容易造成机器负载高 |
| 信号量隔离 | 不支持,如果阻塞，只能通过调用协议（如:socket超时才能返回） | 支持，当信号量达到maxConcurrentRequests后。再请求会触发fallback | 通过信号量的计数器   | 同步调用,不支持异步                   | 小,只是个计数器                             |

```properties
# 修改隔离策略为信号量
hystrix.command.default.execution.isolation.strategy=SEMAPHORE
```
