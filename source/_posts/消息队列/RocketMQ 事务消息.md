---
title: RocketMQ 事务消息
date: {{ date }}
categories:
- 中间件
tags:
- 消息队列
---

指Producer端消息发送事件和本地事务事件，同时成功或同时失败

## 正常事务发送与提交阶段

1. 生产者发送一个半消息给broker(半消息是指的暂时不能消费的消息)
2. 服务端响应
3. 开始执行本地事务
4. 根据本地事务的执行情况执行Commit或者Rollback

## 事务信息的补偿流程

1. 如果broker长时间没有收到本地事务的执行状态,会向生产者发起一个确认回查的操作请求
2. 生产者收到确认会查请求后,检查本地事务的执行状态
3. 根据检查后的结果执行Commit或者Rollback操作，补偿阶段主要是用于解决生产者在发送Commit或者Rollback操作时发生超时或失败的情况

![在这里插入图片描述](https://img-blog.csdnimg.cn/85e09f7030a4401bb70247600d93ccf0.png)

## 事务流程关键

事务消息在一阶段对用户不可见

事务消息相对普通消息最大的特点就是一阶段发送的消息对用户是不可见的,也就是说消费者不能直接消费.这里RocketMQ实现方法是原消息的主题与消息消费队列,然后把主题改成RMQ_SYS_TRANS_HALF_TOPIC.这样由于消费者没有订阅这个主题,所以不会消费.

如何处理第二阶段的发送消息?

在本地事务执行完成后回向Broker发送Commit或者Rollback操作,此时如果在发送消息的时候生产者出故障了,要保证这条消息最终被消费,broker就会向服务端发送回查请求,确认本地事务的执行状态.当然RocketMQ并不会无休止的发送事务状态回查请求,默认是15次,如果15次回查还是无法得知事务的状态,RocketMQ默认回滚消息(broker就会将这条半消息删除)

事务的三种状态

- TransactionStatus.CommitTransaction：提交事务消息，消费者可以消费此消息
- TransactionStatus.RollbackTransaction：回滚事务，它代表该消息将被删除，不允许被消费。
- TransactionStatus.Unknown ：中间状态，它代表需要检查消息队列来确定状态。

## 事务消息约束

1. 事务消息不支持定时和批量
2. 为了避免一个消息被多次检查，导致半数队列消息堆积，RocketMQ限制了单个消息的默认检查次数为15次，通过修改broker配置文件中的transactionCheckMax参数进行调整
3. 特定的时间段之后才检查事务，通过broker配置文件参数transactionTimeout或用户配置CHECK_IMMUNITY_TIME_IN_SECONDS调整时间
4. 一个事务消息可能被检查或消费多次
5. 提交过的消息重新放到用户目标主题可能会失败
6. 事务消息的生产者ID不能与其他类型消息的生产者ID共享

## 事务消息使用

生产者

```java
/**
 * 发送事务消息
 *
 * @author keith
 */
@Slf4j
@RestController
public class TransactionMessageController {

    @Resource
    private RocketMQTemplate rocketMQTemplate;

    @GetMapping("sendTransactionMessage")
    public String sendTransactionMessage() {
        OrderDo orderDo = new OrderDo();
        TransactionSendResult sendResult = rocketMQTemplate.sendMessageInTransaction("tx-topic",
                MessageBuilder.withPayload(orderDo).setHeader("tx_id", orderDo.getOrderId()).build(),
                orderDo
        );
        return sendResult.getTransactionId();
    }
}
```

消费者 -- 接收消息

```java
/**
 * @author keith
 */
@Slf4j
@Component
@RocketMQMessageListener(
        topic = "tx-topic", 					// topic：和生产者发送的topic相同
        consumerGroup = "tx-msg-group",         // group：不用和生产者group相同
        selectorExpression = "*",               // tag
        messageModel = MessageModel.CLUSTERING,
        consumeMode = ConsumeMode.ORDERLY
)
public class OrderTransactionConsumerService implements RocketMQListener<OrderDo> {

    @Override
    public void onMessage(OrderDo orderDo) {
        log.info("接收事务消息结果: {}", orderDo);
    }
}
```

消费者 -- 监听事务消息

```java
@Slf4j
@RocketMQTransactionListener
public class OrderTransactionListenerImpl implements RocketMQLocalTransactionListener {

    @Resource
    private OrderService orderService;

    // 可以将事务状态存到redis，就不会有因为JVM重启后丢失事务状态的问题
    private ConcurrentHashMap<String, RocketMQLocalTransactionState> localTransactions = new ConcurrentHashMap<>();

    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        log.info("执行本地事务, message: {}, arg: {}", msg, arg);
        OrderDo order = (OrderDo) arg;
        try {
            // 执行本地事务
            localTransactions.put(order.getOrderId().toString(), RocketMQLocalTransactionState.UNKNOWN);
            orderService.placeOrder(order);
            localTransactions.put(order.getOrderId().toString(), RocketMQLocalTransactionState.COMMIT);
        } catch (Exception e) {
            // 执行本地事务失败，回滚消息
            log.warn("本地事务执行失败，回滚事务消息，arg: {}", arg, e);
            localTransactions.put(order.getOrderId().toString(), RocketMQLocalTransactionState.COMMIT);
            return RocketMQLocalTransactionState.ROLLBACK;
        }
        // 执行本地事务成功，提交消息
        return RocketMQLocalTransactionState.COMMIT;
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        Object txId = msg.getHeaders().get("tx_id");
        String orderId = txId != null ? txId.toString() : "";
        log.info("检查本地事务, orderId: {}", orderId);

        // 查询本地事务状态
        RocketMQLocalTransactionState transactionState = localTransactions.get(orderId);
        log.info("查询本地事务状态结果: {}", transactionState);

        // 返回本地事务状态
        return transactionState;
    }
}
```

