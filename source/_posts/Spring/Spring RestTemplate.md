---
title: Spring RestTemplate
date: {{ date }}
categories:
- Spring
---

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

## 