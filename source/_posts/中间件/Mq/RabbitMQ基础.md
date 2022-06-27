---
title: RabbitMQ基础
date: {{ date }}
categories:
- 中间件
- MQ
---

## MQ种类

| 特性       | RabbitMQ                                                     | RocketMQ                 | kafka                                                        |
| :--------- | :----------------------------------------------------------- | :----------------------- | :----------------------------------------------------------- |
| 开发语言   | erlang                                                       | java                     | scala                                                        |
| 单机吞吐量 | 万级                                                         | 10万级                   | 10万级                                                       |
| 时效性     | us级                                                         | ms级                     | ms级以内                                                     |
| 可用性     | 高(主从架构)                                                 | 非常高(分布式架构)       | 非常高(分布式架构)                                           |
| 功能特性   | 基于erlang开发，所以并发能力很强，性能极其好，延时很低;管理界面较丰富 | MQ功能比较完备，扩展性佳 | 只支持主要的MQ功能，像一些消息查询，消息回溯等功能没有提供，毕竟是为大数据准备的，在大数据领域应用广。 |

## RabbitMQ工作模型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103225243278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

工作流程：

生产者发送消息给交换机，交换机根据路由规则将消息分发到各个队列中，消费者监听队列，当发现有消息的时候将消息取走。

Channel：信道，多路复用连接中的一条独立的双向数据流通道。信道是建立在真实的TCP连接内的虚拟连接，AMQP命令都是通过信道发送出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说建立和销毁TCP都是非常昂贵的开销，所以引入了信道的概念，以复用一条TCP连接。

VHost：虚拟主机，可以用其分组。提高资源利用率，避免命名的冲突。

Exchange：地址清单，帮助消息分发到各个队列。

Queue：队列，用于存储消息。

## RabbitMQ路由分发规则

### 1. Direct

点对点消息模型，消息中的路由键（routing key）如果和Binding中的绑定键（binding key）一致，交换机就将消息发送到对应的队列中。它是完全匹配、单播的模式。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021010322530492.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 2. Fanout

每个发到fanout类型交换机的消息都会分到所有绑定的队列上去，fanout不处理路由键，它转发消息是最快的，广播模式。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103225322591.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 3.  Topic

通过模式匹配分配消息的路由键属性，将路由键和某个模式匹配，可以识别两个通配符“#”、“*”，有选择性地进行广播。
- \# 匹配0个或多个单词
- \* 匹配一个单词

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103225338676.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## SpringBoot 使用RabbitMQ

### 1. 引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 2. 配置队列、交换机、绑定关系

#### 2.1 配置方式一

##### 2.1.1 队列交换机定义

```java
@Configuration
public class RabbitMqConfigurer {

    @Bean
    public Queue firstQueue(){
        return new Queue("sardine.queue.first");
    }
    
    @Bean
    public DirectExchange directExchange(){
        return new DirectExchange("sardine.exchange.direct");
    }

    @Bean
    public Binding bindFirst(@Qualifier("firstQueue") Queue queue, @Qualifier("directExchange") DirectExchange directExchange){
        return BindingBuilder.bind(queue).to(directExchange).with("demo.first");
    }
}
```

##### 2.1.2 消费者

```java
@Component
@RabbitListener(queues = "sardine.queue.first")
public class FirstConsumer {
    @RabbitHandler
    public void process(String msg){
        System.out.println("rabbitmq receive:" + msg);
    }
}
```

##### 2.1.3 生产者

```java
amqpTemplate.convertAndSend("sardine.exchange.direct", "demo.first", "Hello World");
// 测试结果: rabbitmq receive:Hello World
```

#### 2.2 配置方式二

##### 2.2.1 Direct模式

消费者

```java
@Component
public class WorkConsumer {
    @RabbitListener(queuesToDeclare = @Queue("work"))
    public void receive1(String message){
        System.out.println("message1 = " + message);
    }

    @RabbitListener(queuesToDeclare = @Queue("work"))
    public void receive2(String message){
        System.out.println("message2 = " + message);
    }
}
```

生产者

```java
amqpTemplate.convertAndSend("work", i);
// 测试结果: 平均分配到各个队列
/*
message1 = 0
message2 = 1
message1 = 2
message2 = 3
*/
```

##### 2.2.2 Fanout模式

消费者

```java
@Component
public class FanoutConsumer {
    @RabbitListener(bindings = {
            @QueueBinding(
                    value = @Queue,
                    exchange = @Exchange(value = "fanout.exchange", type = ExchangeTypes.FANOUT)
            )
    })
    public void receive1(String message){
        System.out.println("message1 = " + message);
    }

    @RabbitListener(bindings = {
            @QueueBinding(
                    value = @Queue,
                    exchange = @Exchange(value = "fanout.exchange", type = ExchangeTypes.FANOUT)
            )
    })
    public void receive2(String message){
        System.out.println("message2 = " + message);
    }
}
```

生产者

```java
amqpTemplate.convertAndSend("fanout.exchange","", "Fanout message");
// 测试结果
/*
message1 = Fanout message
message2 = Fanout message
*/
```

##### 2.2.3 Topic模式

消费者

```java
@Component
public class TopicConsumer {
    @RabbitListener(bindings = {
            @QueueBinding(
                    value = @Queue,
                    exchange = @Exchange(name = "topic.exchange", type = ExchangeTypes.TOPIC),
                    key = {"user.save", "user.*"}
            )
    })
    public void receive1(String message){
        System.out.println("message1 = " + message);
    }

    @RabbitListener(bindings = {
            @QueueBinding(
                    value = @Queue,
                    exchange = @Exchange(name = "topic.exchange", type = ExchangeTypes.TOPIC),
                    key = {"order.#", "produce.#", "user.*"}
            )
    })
    public void receive2(String message){
        System.out.println("message2 = " + message);
    }
}
```

生产者

```java
amqpTemplate.convertAndSend("topic.exchange", "user.keith", "Topic user message");
/**
 * 测试结果: 由于两个队列都有user.* 所以都会收到消息
 * message2 = Topic user message
 * message1 = Topic user message
 */
```

```java
amqpTemplate.convertAndSend("topic.exchange", "produce.add.save", "Topic produce message");
/**
 * 测试结果: 由于只有队列2有produce.# 所有只有队列2收到消息
 * message2 = Topic produce message
 */
```

