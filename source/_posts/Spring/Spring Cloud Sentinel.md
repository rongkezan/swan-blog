---
title: Spring Cloud Sentinel
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

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
