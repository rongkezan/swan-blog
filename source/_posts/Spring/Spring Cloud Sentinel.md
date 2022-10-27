---
title: Spring Cloud Sentinel
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

## Sentinel Dashboard

https://github.com/alibaba/Sentinel/wiki/Dashboard

启动 Sentinel Dashboard

```sh
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar
```

登录控制台，默认账号密码都是 `sentinel`

配置控制台信息

```yml
spring:
  cloud:
    sentinel:
      transport:
        port: 8719
        dashboard: localhost:8080
```

## Spring Cloud Sentinel 

pom

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

