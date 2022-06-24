---
title: SpringCloud 服务调用
date: {{ date }}
categories:
- Spring
---

# Spring Cloud 服务调用

## RestTemplate

> springframework:spring-web 包下

使用 RestTemplate 需要将其加入到容器

```java
@Configuration
public class RestTemplateConfigurer {
    @Bean
    public RestTemplate restTemplate(){
        return new restTemplate();
    }
}
```

### 1. 直接调用

实体类

服务端和客户端的实体类可以不是同一个，如果客户端实体类使用@Builder接收会失败

```java
@Data
public class Cookbook {
    private Long id;

    private String bookName;
}
```

消费端

```java
@RestController
@RequestMapping("consumer")
public class ConsumerController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("get")
    public void hello(String name){
        String url = "http://127.0.0.1:8001/provider/hello?name={1}";
        ResponseEntity<String> helloResponse = restTemplate.getForEntity(url, String.class, name);
        System.out.println("Hello: " + helloResponse.getBody());

        String url2 = "http://127.0.0.1:8001/provider/cookbook?name={1}";
        ResponseEntity<Cookbook> cookBookResponse = restTemplate.getForEntity(url2, Cookbook.class, name);
        System.out.println("CookBook: " + cookBookResponse.getBody());

        String url3 = "http://127.0.0.1:8001/provider/list/cookbook";
        List<Cookbook> list = restTemplate.getForObject(url3, List.class);
        System.out.println("CookBook List: " + list);

        String url4 = "http://127.0.0.1:8001/provider/cookbook";
        Student student = new Student();
        student.setName("张三");
        ResponseEntity<Cookbook> postCookbook = restTemplate.postForEntity(url4, student, Cookbook.class);
        System.out.println("Post CookBook: " + postCookbook.getBody());
    }
}
```

服务端

```java
@RestController
@RequestMapping("provider")
public class ProviderController {

    @GetMapping("hello")
    public String hello(String name){
        return "Hello " + name;
    }

    @GetMapping("cookbook")
    public Cookbook cookbook(String name){
        return Cookbook.builder().id(1L).bookName(name).build();
    }

    @GetMapping("list/cookbook")
    public List<Cookbook> listCookbook(){
        List<Cookbook> list = new ArrayList<>();
        list.add(Cookbook.builder().id(1L).bookName("盐酥鸡").build());
        list.add(Cookbook.builder().id(2L).bookName("酱鸭").build());
        return list;
    }

    @PostMapping("cookbook")
    public Cookbook postCookbook(@RequestBody Student student){
        return Cookbook.builder().bookName(student.getName()).build();
    }
}
```

### 2. 注册到同一个 eureka 

消费端

```java
@RestController
@RequestMapping("consumer")
public class ConsumerController {

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private DiscoveryClient discoveryClient;

    @GetMapping("hello")
    public void hello(String name){
        //拿到服务提供商
        List<ServiceInstance> list = discoveryClient.getInstances("sardine-cookbook");
        //拿到第一个实例
        ServiceInstance instance = list.get(0);
        //得到主机号
        String host = instance.getHost();
        //得到端口号
        int port = instance.getPort();
        //拼接完整的请求url
        String url = "http://" + host + ":" + port + "/test/hello?name={1}";
        //restTemple 实际返回的是一个ResponseEntity 的实例
        ResponseEntity<String> responseEntity = restTemplate.getForEntity(url, String.class, name);
        System.out.println(responseEntity.getBody());
    }
}
```

服务端

```java
@RestController
@RequestMapping("provider")
public class ProviderController {

    @GetMapping("hello")
    public String hello(String name){
        return "Hello " + name;
    }
}
```

### 3. 客户端拦截器

```java
public class LoggingClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        System.out.println("执行了拦截器");
        System.out.println(request.getURI());
        ClientHttpResponse response = execution.execute(request, body);
        return response;
    }
}
```

在 restTemplate 配置中增加拦截器

```java
@Configuration
public class RestTemplateConfigurer {
    @Bean
    public RestTemplate restTemplate(){
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add(new LoggingClientHttpRequestInterceptor());
        return restTemplate;
    }
}
```

## Ribbon

> 客户端的负载均衡

Ribbon是Netflix开发的客户端负载均衡器，为Ribbon配置**服务提供者地址列表**后，Ribbon就可以基于某种**负载均衡策略算法**，自动地帮助服务消费者去请求提供者。Ribbon默认为我们提供了很多负载均衡算法，例如轮询、随机等。我们也可以实现自定义负载均衡算法。

### 1. 负载均衡算法

- **ZoneAvoidanceRule（默认）：区域权衡策略**

  复合判断Server所在区域的性能和Server的可用性，轮询选择服务器。

- **BestAvailableRule：最低并发策略**

  会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务。逐个找服务，如果断路器打开，则忽略。

- **RoundRobinRule：轮询策略**

  以简单轮询选择一个服务器，按顺序循环选择一个server。

- **RandomRule：随机策略**

  随机选择一个服务器。

- **AvailabilityFilteringRule：可用过滤策略**

  会先过滤掉多次访问故障而处于断路器跳闸状态的服务和过滤并发的连接数量超过阀值得服务，然后对剩余的服务列表安装轮询策略进行访问

- **WeightedResponseTimeRule：响应时间加权策略**

  据平均响应时间计算所有的服务的权重，响应时间越快服务权重越大，容易被选中的概率就越高。刚启动时，如果统计信息不中，则使用RoundRobinRule(轮询)策略，等统计的信息足够了会自动的切换到WeightedResponseTimeRule。响应时间长，权重低，被选择的概率低。反之，同样道理。此策略综合了各种因素（网络，磁盘，IO等），这些因素直接影响响应时间。

- **RetryRule：重试策略**

  先按照RoundRobinRule(轮询)的策略获取服务，如果获取的服务失败则在指定的时间会进行重试，进行获取可用的服务。如多次获取某个服务失败，就不会再次获取该服务。主要是在一个时间段内，如果选择一个服务不成功，就继续找可用的服务，直到超时。

开启负载均衡

```java
@Bean
@LoadBalanced
RestTemplate restTemplate() {
	return new RestTemplate();
}
```

切换策略

```java
@Bean
public IRule myRule() {
    //return new RoundRobinRule();
    //return new RandomRule();
    return new RetryRule();
}
```

消费端：注意要使用应用名替代域名，且要在同一个注册中心，否则会报错

```java
@GetMapping("balance")
public void balance(){
    String url = "http://sardine-cookbook/provider/balance";
    ResponseEntity<String> response = restTemplate.getForEntity(url, String.class);
    System.out.println(response.getBody());
}
```

服务端

```java
@GetMapping("balance")
public String balance(){
    return "Hello World";
}
```

### 2. 配置

```properties
# 请求连接的超时时间
ribbon.ConnectTimeout=2000
# 请求处理的超时时间
ribbon.ReadTimeout=5000
# 也可以为每个Ribbon客户端设置不同的超时时间, 通过服务名称进行指定：
ribbon-config-demo.ribbon.ConnectTimeout=2000
ribbon-config-demo.ribbon.ReadTimeout=5000
# 最大连接数
ribbon.MaxTotalConnections=500
# 每个host最大连接数
ribbon.MaxConnectionsPerHost=500
```

**ribbon脱离eureka配置**

可以在配置文件中使用listOfServers字段来设置服务端地址，只要客户端拥有服务器列表，就可以使用ribbon做负载均衡。

```properties
ribbon.eureka.enabled=false
ribbon.listOfServers=localhost:8001,localhost:8002
```

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

## Sentinel

### 简单使用

pom

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.2.5.RC2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
</dependencies>
```

```java
@SpringBootApplication
public class SardineAlibabaSamplesApplication {

    public static void main(String[] args) {
        initSentinelRules();
        SpringApplication.run(SardineAlibabaSamplesApplication.class, args);
    }

    public static void initSentinelRules(){
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule();
        rule.setResource("resource");
        // set limit qps to 2
        rule.setCount(2);
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setLimitApp("default");
        rules.add(rule);
        FlowRuleManager.loadRules(rules);
    }
}
```

```java
@RestController
public class TestController {

   @GetMapping("hello")
   @SentinelResource(value = "resource", blockHandler = "fallback")
   public String hello() {
      return "Hello";
   }

   public String fallback(BlockException e){
      return "降级了";
   }
}
```

调用 hello 接口，若qps大于2的时候会执行降级方法。

### 整合 Sentinel Dashboard

https://github.com/alibaba/Sentinel/wiki/Dashboard

启动 Sentinel Dashboard，登录控制台，默认账号密码都是 sentinel

**在服务中增加以下配置以注册到Dashboard**

```properties
server.port=8000
spring.cloud.sentinel.transport.dashboard=localhost:8080
# 服务启动后就注册到 Dashboard
spring.cloud.sentinel.eager=true
spring.application.name=stn
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210211094241995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

新增一条流控规则

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210211095141129.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)
