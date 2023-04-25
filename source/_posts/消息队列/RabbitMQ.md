---
title: RabbitMQ
date: {{ date }}
categories:
- 中间件
tags:
- 消息队列
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

## 过期队列

### 概念

过期时间TTL表示可以对消息设置预期的时间，在这个时间内都可以被消费者接收，过了之后消息自动删除。

设置消息和队列的两种方式

- 通过队列属性设置，队列中所有消息都有相同的过期时间
- 对消息单独设置，每条消息的TTL可以不同

以上两种方法如果同时使用，则消息的过期时间以两者之间TTL较小的为准。消息在队列的生存时间一旦超过设置的TTL值，就称为dead message 被投递到死信队列，消费者无法再收到该消息。

### Spring Boot 配置

#### xml 配置

```xml
<!-- 定义过期队列及属性 不存在则自动创建 -->
<rabbit:queue id="myTtlQueue" name="my.ttl.queue" auto-declare="true">
    <rabbit:queue-arguments>
        <!-- 投递到该队列的消息如果没有消费将在6秒后被删除 -->
        <entry key="x-message-ttl" value-type="long" value="6000" />
    </rabbit:queue-arguments>
</rabbit:queue>

<!-- 定义定向交换机，根据不同的路由key投递信息 -->
<rabbit:direct-exchange id="myDirExchange" name="my.dir.exchange" auto-declare="true">
    <rabbit:bindings>
        <rabbit:binding key="my.ttl.dlx" queue="my.ttl.queue" />
    </rabbit:bindings>
</rabbit:direct-exchange>
```

#### 测试

消息在6秒内未被消费则会过期

```java
amqpTemplate.convertAndSend("my.dir.exchange", "my.ttl.dlx", "Hello World");
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103224630359.png)

### 单个消息配置过期时间

```java
MessageProperties messageProperties = new MessageProperties();
messageProperties.setExpiration("5000");
Message message = new Message("This message will expired in 5s".getBytes(), messageProperties);
amqpTemplate.convertAndSend("sardine.exchange.direct", "demo.first", message);
```

## 死信队列

### 概念

DLX，全称为Dead Letter Exchange，可以称为死信交换机。当消息在一个队列中变成死信（dead message）之后，它能被重新发送到另一个交换机中，这个交换机就是DLX，绑定DLX的队列就称之为死信队列。

消息变成死信，可能的原因：

- 消息被拒绝
- 消息过期
- 队列达到最大长度

DLX也是一个正常的交换机，和一般的交换机没什么区别，它能在任何队列上被指定，实际上就是设置某一个队列的属性。当这个队列中存在死信时，Rabbitmq就会自动地将这个消息重新发布到设置的DLX上去，进而被路由到另一个队列，即死信队列。

要想使用死信队列，只需要在定义队列的时候设置队列参数 `x-dead-letter-exchange` 指定交换机即可。

### 图解

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103134303902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### Spring Boot 配置

#### xml配置

消息发送到 `定向交换机` ，6秒后消息过期，转发到 `死信交换机`，死信交换机再根据路由键转发到 `死信队列`

```xml
<!-- 定义过期队列及属性 不存在则自动创建 -->
<rabbit:queue id="myTtlQueue" name="my.ttl.queue" auto-declare="true">
    <rabbit:queue-arguments>
        <!-- 投递到该队列的消息如果没有消费将在6秒后被删除 -->
        <entry key="x-message-ttl" value-type="long" value="6000" />
        <!-- 设置当消息过期后投递到对应的死信交换机 -->
        <entry key="x-dead-letter-exchange" value="my.dlx.exchange" />
    </rabbit:queue-arguments>
</rabbit:queue>

<!-- 定义定向交换机，根据不同的路由key投递信息 -->
<rabbit:direct-exchange id="myDirExchange" name="my.dir.exchange" auto-declare="true">
    <rabbit:bindings>
        <rabbit:binding key="my.ttl.dlx" queue="myTtlQueue" />
    </rabbit:bindings>
</rabbit:direct-exchange>

<!-- 定义死信队列 -->
<rabbit:queue id="myDlxQueue" name="my.dlx.queue" auto-declare="true" />

<!-- 定义死信交换机 -->
<rabbit:direct-exchange id="myDlxExchange" name="my.dlx.exchange" auto-declare="true">
    <rabbit:bindings>
        <!-- 过期的消息转移到my.dlx.queue队列 -->
        <rabbit:binding key="my.ttl.dlx" queue="myDlxQueue" />
    </rabbit:bindings>
</rabbit:direct-exchange>
```

#### 测试

当消息过期后，会被移动到死信队列中

```java
amqpTemplate.convertAndSend("my.dir.exchange", "my.ttl.dlx", "Hello World");
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103224657166.png)

## 延迟队列

延迟队列存储的对象是对应的延迟信息，所谓"延迟信息"是指消息被发送后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。

在RabbitMQ中延迟队列可以通过 `过期时间 + 死信队列`来实现

应用场景：电商中的支付场景，如果用户下单之后的几十分钟都没有支付成功，那个这个支付的订单则记为支付失败，要进行支付失败的异常处理（将库存加回去），这个时候可以通过延迟队列来实现。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103134331250.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 消息确认机制

确认并保证消息送达，提供了两种方式：发布确认和事务（两者不可同时使用）。

发布确认的两种方式：消息发送成功确认，消息发送失败回调

**开启手动确认模式**

添加配置

```yml
publisher-confirms: true        # 开启Confirm确认机制
publisher-returns: true         # 开启Return确认机制
template:
  mandatory: true               # 消费者在消息没有被路由到合适的队列下会被return监听，不会自动删除
listener:
  simple:
    acknowledge-mode: manual    # 消费端手动ACK
    retry:
      enabled: true             # 是否支持重试
```

消息确认处理类

```java
@Slf4j
@Component
public class MessageConfirmCallback implements RabbitTemplate.ConfirmCallback {

    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        System.out.println("correlationData = " + correlationData);
        System.out.println("ack = " + ack);
        if (!ack){
            log.error("消息处理失败:{}", cause);
        }
    }
}
```

消息返回处理类

```java
@Slf4j
@Component
public class MessageReturnCallback implements RabbitTemplate.ReturnCallback {

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("returned Message ===> message={}, replyCode={} ,replyText={} ,exchange={} ,routingKey={}", message, replyCode, replyText, exchange, routingKey);
    }
}
```

消息发送方

```java
@Component
public class SardineRabbitSender {

    @Resource
    RabbitTemplate rabbitTemplate;

    @Resource
    MessageConfirmCallback messageConfirmCallback;

    @Resource
    MessageReturnCallback messageReturnCallback;

    public void send(String exchange, String routingKey, Object message) {
        rabbitTemplate.setConfirmCallback(messageConfirmCallback);
        rabbitTemplate.setReturnCallback(messageReturnCallback);
        rabbitTemplate.convertAndSend(exchange, routingKey, message, msg -> {
            msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
            return msg;
        }, new CorrelationData(UUID.randomUUID().toString()));
    }
}
```

消息接收方

```java
@Slf4j
@Component
public class SardineRabbitReceiver {
    @RabbitListener(
            bindings = @QueueBinding(
                    value = @Queue(value = "boot.queue"),
                    exchange = @Exchange(value = "boot.exchange", type = ExchangeTypes.TOPIC),
                    key = "boot.*"
            )
    )
    public void onMessage(Message message, Channel channel) throws IOException {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            channel.basicAck(deliveryTag,false);
            System.out.println("收到消息: " + new String(message.getBody()));
        } catch (Exception e) {
            log.error("消息处理失败", e);
            if (message.getMessageProperties().getRedelivered()){
                log.error("消息已重复处理失败，拒绝再次接收");
                channel.basicReject(deliveryTag, false);
            } else {
                log.warn("消息即将再次返回队列处理");
                // 消息处理失败，重新入队
                channel.basicNack(deliveryTag, false, true);
            }
        }
    }
}
```

测试类

```java
sardineRabbitSender.send("boot.exchange", "boot.queue","Hello Spring Boot");
```

测试结果（消息确认： `channel.basicAck(deliveryTag,false)`）

```
correlationData = CorrelationData [id=57808143-1800-4e0c-8405-f6e566fa06f1]
ack = true
收到消息: Hello Spring Boot
```

测试结果（消息不确认：没有 `channel.basicAck(deliveryTag,false)`）

```java
收到消息: Hello Spring Boot
correlationData = CorrelationData [id=0e4890f3-9b02-44e1-8dfc-49b4463415c2]
ack = false
```

## 消息追踪

Docker环境下查看和启用rabbitmq消息追踪

```powershell
# 查看rabbitmq所有插件
docker exec -it rabbitmq rabbitmq-plugins list
# 启用rabbitmq_tracing
docker exec -it rabbitmq rabbitmq-plugins enable rabbitmq_tracing
# 打开rabbitmq消息追踪
docker exec -it rabbitmq rabbitmqctl trace_on
# 设置消息追踪的VHost
docker exec -it rabbitmq rabbitmqctl trace_on -p sardine

# 关闭rabbitmq消息追踪
docker exec -it rabbitmq rabbitmqctl trace_off
# 只有administrator角色才能查看日志界面
docker exec -it rabbitmq rabbitmqctl set_user_tags admin administrator
```

开启之后发现管理界面多了一个交换机 `amp.rabbitmq.trace`

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021010316423370.png)

在admin界面中创建一个新的trace

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103164544972.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

rabbitmq每发送一次消息都会往日志中写入，所以是比较消耗性能的，只在开发调试中使用

日志格式（TEXT）

```tex
Node:         rabbit@node1
Connection:   <rabbit@node1.3.3552.0>
Virtual host: /
User:         root
Channel:      1
Exchange:     exchange
Routing keys: [<<"rk">>]
Routed queues: [<<"queue">>]
Properties:   [{<<"delivery_mode">>,signedint,1},{<<"headers">>,table,[]}]
Payload: 
trace test payload.
```

