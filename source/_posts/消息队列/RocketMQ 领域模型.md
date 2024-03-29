---
title: RocketMQ 领域模型
date: {{ date }}
categories:
- 消息队列
---

![领域模型](https://rocketmq.apache.org/zh/assets/images/mainarchi-9b036e7ff5133d050950f25838367a17.png)

如上图所示，Apache RocketMQ 中消息的生命周期主要分为消息生产、消息存储、消息消费这三部分。

生产者生产消息并发送至 Apache RocketMQ 服务端，消息被存储在服务端的主题中，消费者通过订阅主题消费消息。

**消息生产**

[生产者（Producer）](https://rocketmq.apache.org/zh/docs/domainModel/04producer)：

Apache RocketMQ 中用于产生消息的运行实体，一般集成于业务调用链路的上游。生产者是轻量级匿名无身份的。

**消息存储**

- [主题（Topic）](https://rocketmq.apache.org/zh/docs/domainModel/02topic)：

  Apache RocketMQ 消息传输和存储的分组容器，主题内部由多个队列组成，消息的存储和水平扩展实际是通过主题内的队列实现的。

- [队列（MessageQueue）](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue)：

  Apache RocketMQ 消息传输和存储的实际单元容器，类比于其他消息队列中的分区。 Apache RocketMQ 通过流式特性的无限队列结构来存储消息，消息在队列内具备顺序性存储特征。

- [消息（Message）](https://rocketmq.apache.org/zh/docs/domainModel/04message)：

  Apache RocketMQ 的最小传输单元。消息具备不可变性，在初始化发送和完成存储后即不可变。

**消息消费**

- [消费者分组（ConsumerGroup）](https://rocketmq.apache.org/zh/docs/domainModel/07consumergroup)：

  Apache RocketMQ 发布订阅模型中定义的独立的消费身份分组，用于统一管理底层运行的多个消费者（Consumer）。同一个消费组的多个消费者必须保持消费逻辑和配置一致，共同分担该消费组订阅的消息，实现消费能力的水平扩展。

- [消费者（Consumer）](https://rocketmq.apache.org/zh/docs/domainModel/08consumer)：

  Apache RocketMQ 消费消息的运行实体，一般集成在业务调用链路的下游。消费者必须被指定到某一个消费组中。

- [订阅关系（Subscription）](https://rocketmq.apache.org/zh/docs/domainModel/09subscription)：

  Apache RocketMQ 发布订阅模型中消息过滤、重试、消费进度的规则配置。订阅关系以消费组粒度进行管理，消费组通过定义订阅关系控制指定消费组下的消费者如何实现消息过滤、消费重试及消费进度恢复等。

  Apache RocketMQ 的订阅关系除过滤表达式之外都是持久化的，即服务端重启或请求断开，订阅关系依然保留。