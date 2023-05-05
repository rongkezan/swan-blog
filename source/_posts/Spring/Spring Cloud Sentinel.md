---
title: Spring Cloud Sentinel
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

## Sentinel 概述

> 分布式系统的流量防卫兵

随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel 以流量为切入点，从流量控制、流量路由、熔断降级、系统自适应过载保护、热点流量防护等多个维度保护服务的稳定性。

Sentinel 的主要特性：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2612e1d553a448d688768a4eea284b60.png)

## Sentinel 滑动窗口

窗口：你可以这么理解，一段是时间就是窗口，比如说我们可以把这个1s认为是1个窗口

滑动：滑动很简单，比如说现在时间由第1秒变成了第2秒，就是从当前这个窗口---->下一个窗口就可以了，这个时候下一个窗口就变成了当前窗口，之前那个当前窗口就变成了上一个窗口，这个过程其实就是滑动。

**先是介绍下窗口中里面都存储些啥**

- 它得有个开始时间吧，不然你怎么知道这个窗口是什么时候开始的
- 还得有个窗口的长度吧，不然你咋知道窗口啥时候结束，通过这个开始时间+窗口长度=窗口结束时间，就比如说上面的1s，间隔1s
- 最后就是要在这个窗口里面统计的东西，你总不能白搞些窗口，搞些滑动吧。所以这里就存储了一堆要统计的指标（qps，rt等等）

**说完了这一个小窗口里面的东西，就得来说说是怎么划分这个小窗口，怎么管理这些小窗口的了。**

- 要知道有多少个小窗口，在sentinel中也就是sampleCount，比如说我们有60个窗口。
- 还有就是intervalInMs，这个intervalInMs是用来计算这个窗口长度的，intervalInMs/窗口数量= 窗口长度。也就是我给你1分钟，你给我分成60个窗口，这个时候窗口长度就是1s了，那如果我给你1s，你给我分2个窗口，这个时候窗口长度就是500毫秒了，这个1分钟，就是intervalInMs。
- 再就是存储这个窗口的容器（这里是数组），毕竟那么多窗口，还得提供计算当前时间窗口的方法等等

**最后我们就来看看这个当前时间窗口是怎么计算的。**

咱们就拿 60个窗口，这个60个窗口放在数组中，窗口长度是1s 来计算，看看当前时间戳的一个时间窗口是是在数组中哪个位置。比如说当前时间戳是1609085401454 ms

算出秒 = 1609085401454 /1000（窗口长度）

在数组的位置 = 算出秒 % 数组长度 =

## Sentinel Dashboard

Wiki: https://github.com/alibaba/Sentinel/wiki/Dashboard

Download: https://github.com/alibaba/Sentinel/releases

启动 Sentinel Dashboard

```sh
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar
```

登录控制台，默认账号密码都是 `sentinel`

## Spring Boot 整合 Sentinel 

引入依赖

```xml
<dependencyManagement>
    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.3.5.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- Spring Cloud -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR12</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- Spring Cloud Alibaba -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.2.5.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
        <!-- Spring Cloud Alibaba Nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
    
        <!-- Spring Cloud Alibaba Sentinel -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>

        <!-- Spring Cloud Sentinel Datasource Nacos  -->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
</dependencies>
```

配置控制台信息

```yml
spring:
  cloud:
    sentinel:
      transport:
        port: 8719
        dashboard: localhost:8080
```

这里的 `spring.cloud.sentinel.transport.port` 端口配置会在应用对应的机器上启动一个 Http Server，该 Server 会与 Sentinel 控制台做交互。比如 Sentinel 控制台添加了一个限流规则，会把规则数据 push 给这个 Http Server 接收，Http Server 再将规则注册到 Sentinel 中。

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
