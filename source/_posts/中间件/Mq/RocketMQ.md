---
title: RocketMQ
date: {{ date }}
categories:
- 中间件
- MQ
---

> http://rocketmq.apache.org

## 安装

### Server

下载最新的 binary release，修改配置（原因是默认配置消耗太多内存）

```shell
vi runserver.sh
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"

vi runbroker.sh
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn128m"
```

启动/关闭 RocketMQ

```shell
# Start Name Server
nohup sh bin/mqnamesrv &
# Start Broker
nohup sh bin/mqbroker -n localhost:9876 &

# Shutdown Name Server
sh bin/mqshutdown namesrv
# Shutdown Broker
sh bin/mqshutdown broker
```

查看启动日志

```sh
# 查看 namesrv 的启动日志
tail -f ~/logs/rocketmqlogs/namesrv.log
# 查看 broker 的启动日志
tail -f ~/logs/rocketmqlogs/broker.log
```

测试

```shell
# 消费端
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer

# 服务端
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

### Console

下载 rocketmq-externals：https://github.com/apache/rocketmq-externals

解压后修改 rocketmq-console 模块中的 application.properties

```properties
rocketmq.config.namesrvAddr=localhost:9876
```

在 rocketmq-console 目录下编译

```shell
mvn clean package -Dmaven.test.skip=true
```

运行编译完的 jar 包

```shell
java -jar rocketmq-console-ng-1.0.0.jar --server.port=12581
```

## 基本概念

### 角色

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210301111718407.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

#### Broker

- Broker 面向 Producer 和 Consumer 接受和发送消息
- 向 Nameserver 提交自己的信息
- 是消息中间件的消息存储、转发服务器。
- 每个Broker节点，在启动时，都会遍历NameServer列表，与每个NameServer建立长连接，注册自己的信息，之后定时上报。

**Broker集群**

- Broker高可用，可以配成Master/Slave结构，Master可写可读，Slave只可以读，Master将写入的数据同步给Slave。
  - 一个Master可以对应多个Slave，但是一个Slave只能对应一个Master
  - Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义BrokerId为0表示Master，非0表示Slave
- Master多机负载，可以部署多个broker
  - 每个Broker与nameserver集群中的所有节点建立长连接，定时注册Topic信息到所有nameserver。

#### Group

消费者可能存在多个，一个消费者集群就代表一个 Group，消息投递到 Group 后只要某一个消费者消费了，就算成功消费，即消息的消费是以 Group 为单位的。

#### Producer

- 消息的生产者
- 通过集群中的其中一个节点（随机选择）建立长连接，获得Topic的路由信息，包括Topic下面有哪些Queue，这些Queue分布在哪些Broker上等
- 接下来向提供Topic服务的Master建立长连接，且定时向Master发送心跳

#### Consumer

消息的消费者，通过NameServer集群获得Topic的路由信息，连接到对应的Broker上消费消息。

注意，由于Master和Slave都可以读取消息，因此Consumer会与Master和Slave都建立连接。

#### Nameserver

底层由netty实现，提供了路由管理、服务注册、服务发现的功能，是一个无状态节点

**nameserver是服务发现者**，集群中各个角色（producer、broker、consumer等）都需要定时想nameserver上报自己的状态，以便互相发现彼此，超时不上报的话，nameserver会把它从列表中剔除

**nameserver可以部署多个**，当多个nameserver存在的时候，其他角色同时向他们上报信息，以保证高可用，

**NameServer集群间互不通信**，没有主备的概念

**nameserver内存式存储**，nameserver中的broker、topic等信息默认不会持久化

**为什么不用zookeeper？**：rocketmq希望为了提高性能，CAP定理，客户端负载均衡

### Topic

Topic是一个逻辑上的概念，实际上Message是在每个Broker上以Queue的形式记录。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021030118144581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## Demo Code

> 消息的发送和接收

### 同步发送接收消息

生产者

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQProducer producer = new DefaultMQProducer("MyGroup");
        // Set nameServer address
        producer.setNamesrvAddr("192.168.25.101:9876");
        producer.start();
        producer.setSendMsgTimeout(6000);
        // Topic: The address to which message will be sent
        // Body: The real message
        Message message = new Message("myTopic", "myMessage".getBytes());
        // Synchronized send, it will blocking, slow but don't lose message
        SendResult result = producer.send(message);
        // Get consumer's fallback
        System.out.println(result);
        producer.shutdown();
    }
}
```

消费者

```java
public class Consumer {
    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("MyConsumer");
        consumer.setNamesrvAddr("192.168.25.101:9876");
        // Each consumer just focus one topic
        // Topic: the address to which message will be sent
        // Filter: * refers to not filter
        consumer.subscribe("myTopic", "*");
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    String msgStr = new String(msg.getBody());
                    System.out.println(msgStr);
                }
                // This message will be consumed by one consumer in default situation, point to point.
                // On the other hand, consumer can update status of message, ack
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.println("Consumer started.");
    }
}
```

### 生产者常用的发送模式

1. 只发送消息，不考虑消息是否成功消费、

```java
Message message = new Message("myTopic", "myOnewayMessage".getBytes());
producer.sendOneway(message);
```

2. 同步发送消息，可靠，阻塞

```java
Message message = new Message("myTopic", "myMessage".getBytes());
SendResult result = producer.send(message);
```

3. 异步发送消息，可靠，不阻塞

```java
Message message = new Message("myTopic", "myAsyncMessage".getBytes());
producer.send(message, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        System.out.println("Send message success, sendResult:" + sendResult);
    }

    @Override
    public void onException(Throwable e) {
        // You can catch this exception, do some compensation, retry maybe.
        System.out.println("Send message exception, e:" + e.getMessage());
    }
});
```

4. 发送带有 TAG、KEY 的消息

```java
Message message = new Message("myTopic", "MyTag", "MyKey", "myMessage".getBytes());
SendResult result = producer.send(message);
```

## SQL 表达式过滤

消费者将收到包含TAGA或TAGB或TAGB的消息. 但限制是一条消息只能有一个标签，而这对于复杂的情况可能无效。 在这种情况下，您可以使用SQL表达式筛选出消息.

在`broker.conf `中添加配置

```shell
enablePropertyFilter=true
```

启动 broker 加载指定配置文件

```shell
../bin/mqbroker -n 192.168.150.113:9876 -c broker.conf 
```

随后在集群配置中可以看到

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210302171804846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

代码

```java
MessageSelector selector = MessageSelector.bySql("order > 5");
consumer.subscribe("myTopic", selector);
```

## 事务消息

1. Half Message：预处理消息，当broker收到此类消息后，会存储到 RMQ_SYS_TRANS_HALF_TOPIC 的消息消费队列中

2. 检查事务状态：Broker会开启一个定时任务，消费RMQ_SYS_TRANS_HALF_TOPIC队列中的消息，每次执行任务会向消息发送者确认事务执行状态（提交、回滚、未知），如果是未知，等待下一次回调。

3. 超时：如果超过回查次数，默认回滚消息

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303135132451.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

```java
public class ProducerTransaction {
    public static void main(String[] args) throws MQClientException {
        TransactionMQProducer producer = new TransactionMQProducer("MyTransactionGroup");
        producer.setNamesrvAddr("192.168.25.101:9876");
        producer.setSendMsgTimeout(6000);
        producer.setTransactionListener(new TransactionListener() {
            @Override
            public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
                // Execute local transaction
                try {
                    System.out.println("--- Execute ---");
                    System.out.println("msg: " + new String(msg.getBody()));
                    System.out.println("msg: " + msg.getTransactionId());
                    return LocalTransactionState.COMMIT_MESSAGE;
                } catch (Exception e) {
                    return LocalTransactionState.ROLLBACK_MESSAGE;
                }
            }

            @Override
            public LocalTransactionState checkLocalTransaction(MessageExt msg) {
                // When no response to half message, this method will be invoked.
                System.out.println("--- Check ---");
                System.out.println("msg: " + new String(msg.getBody()));
                System.out.println("msg: " + msg.getTransactionId());
                return LocalTransactionState.UNKNOW;
            }
        });
        producer.start();
        producer.sendMessageInTransaction(new Message("myTopic", "Test Transaction Message".getBytes()), null);
    }
}
```

本地事务执行状态

```java
// 执行事务成功，确认提交
LocalTransactionState.COMMIT_MESSAGE;
// 回滚消息，broker端会删除半消息
LocalTransactionState.ROLLBACK_MESSAGE;
// 暂时为未知状态，等待broker回查
LocalTransactionState.UNKNOW;
```

## 重试机制

生产者

```java
// 异步发送时 重试次数，默认 2
producer.setRetryTimesWhenSendAsyncFailed(1);
// 同步发送时 重试次数，默认 2
producer.setRetryTimesWhenSendFailed(1);	
// 是否向其他broker发送请求 默认false
producer.setRetryAnotherBrokerWhenNotStoreOK(true);
```
消费者

```java
// set consume timeout (minute)
consumer.setConsumeTimeout(1);
```

```java
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
	// return status 'RECONSUME_LATER',it will retry to consume this message.
    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
});
```

**Broker投递**

只有在消息模式为集群模式（MessageModel.CLUSTERING）时，Broker才会自动进行重试，广播消息不重试

重投使用`messageDelayLevel`

默认值

```
messageDelayLevel 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

## 消息重复消费

**引起重复消费的原因**

ACK：正常情况下 Consumer 消费完消息之后会发送 ACK 通知 Broker，Broker 从队列中剔除这条消息，当 ACK 因为网络原因无法发送到 Broker，Broker 会认为该消息没有被消费，消息重投机制会把消息再次投递。

Group：在集群模式下，消息在 Broker 中会保证相同 Group 的 Consumer 消费一次，但针对不同 Group 的 Consumer 会推送多次。

**解决方案**

数据库：处理消息前，使用消息表主键带有约束字段中 insert

Map（单机）：putIfAbsent

Redis：分布式锁

## 顺序消费

**生产者**

使用 MessageQueueSelector 的实现

```java
public interface MessageQueueSelector {
    MessageQueue select(final List<MessageQueue> mqs, final Message msg, final Object arg);
}
```

SelectMessageQueueByHash：通过第三个参数判断发送到哪个队列

```java
// Same arg message will send to same queue.
producer.send(message, new SelectMessageQueueByHash(), 1);
```

**消费者**

```java
// MessageListenerConcurrently: 并发消费，会开启多个线程
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) 
                                 -> ConsumeConcurrentlyStatus.CONSUME_SUCCESS);
// MessageListenerOrderly: 顺序消费，一个队列开启一个线程
consumer.registerMessageListener((MessageListenerOrderly) (msgs, context) 
                                 -> ConsumeOrderlyStatus.SUCCESS);
// 设置最大线程数
consumer.setConsumeThreadMax(1);
// 设置最小线程数
consumer.setConsumeThreadMin(1);
```

