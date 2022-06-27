---
title: Kafka
date: {{ date }}
categories:
- 中间件
- MQ
---

Kafka配置文件 server.properties

```shell
broker.id=0
listeners=PLAINTEXT://node01:9092
log.dirs=/var/kafka_data
zookeeper.connect=node02:2181,node03:2181,node04:2181/kafka
```

 