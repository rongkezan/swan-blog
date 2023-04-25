---
title: RocketMQ 整合 SpringBoot
date: {{ date }}
categories:
- 中间件
tags:
- 消息队列
---

依赖引入

```xml
<!-- Spring Boot Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>2.4.2</version>
</dependency>

<!-- RocketMQ -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.2</version>
</dependency>
```

配置文件

```yaml
server:
  port: 12580
spring:
  application:
    name: sardine-rocketmq
rocketMq:
  name-server: 127.0.0.1:9876
  producer:
    group: ${spring.application.name}       # 发送同一类消息设置为同一个Group，保证唯一
    send-message-timeout: 10000             # 发送消息的超时时间，默认3000
    retry-times-when-send-failed: 2         # 发送消息失败重试次数，默认2
    retry-times-when-send-async-failed: 2   # 发送异步消息失败重试次数，默认2
    max-message-size: 4096                  # 消息的最大长度，默认 1024*1024*4 = 4M
    compress-message-body-threshold: 4096   # 压缩消息阈值，默认4K
    retry-next-server: false                # 是否在内部发送失败时重试另一个Broker，默认false
```

重写 rocketMQMessageConverter

> RocketMQ内置使用的转换器是**RocketMQMessageConverter**，转换Json时使用的是MappingJackson2MessageConverter，但是这个转换器不支持时间类型。所以转换LocalDateTime时会报错

```java
@Configuration
public class RocketMQConfig {

    @Bean
    @Primary
    public RocketMQMessageConverter rocketMQMessageConverter(){
        RocketMQMessageConverter converter = new RocketMQMessageConverter();
        CompositeMessageConverter compositeMessageConverter = (CompositeMessageConverter) converter.getMessageConverter();
        List<MessageConverter> messageConverters = compositeMessageConverter.getConverters();
        for (MessageConverter messageConverter : messageConverters) {
            if (messageConverter instanceof MappingJackson2MessageConverter) {
                MappingJackson2MessageConverter mappingJackson2MessageConverter = (MappingJackson2MessageConverter) messageConverter;
                ObjectMapper objectMapper = mappingJackson2MessageConverter.getObjectMapper();
                objectMapper.registerModule(new JavaTimeModule());
            }
        }
        return converter;
    }
}
```

