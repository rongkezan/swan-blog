---
title: RocketMQ 安装运行
date: {{ date }}
categories:
- 中间件
tags:
- 消息队列
---

## 安装

### Server

下载最新的 binary release，修改配置（原因是默认配置消耗太多内存）

```shell
vi runserver.sh
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"

vi runbroker.sh
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn128m"
```

Linux 启动/关闭 RocketMQ

```shell
# Start Name Server
nohup sh mqnamesrv &
# Start Broker
nohup sh mqbroker -n localhost:9876 &

# Shutdown Name Server
sh mqshutdown namesrv
# Shutdown Broker
sh mqshutdown broker
```

Windows 启动 RocketMQ

```sh
# Start Name Server
start mqnamesrv.cmd
# Start Broker
start mqbroker.cmd -n localhost:9876
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
# 生产者
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer

# 消费者
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

### Dashboard

下载RocketMQ-Dashboard https://github.com/apache/rocketmq-dashboard/releases

解压后修改 rocketmq-dashboard 模块中的 application.properties

```properties
rocketmq.config.namesrvAddr=localhost:9876
server.port=12581
```

在 rocketmq-dashboard 目录下编译

```shell
mvn clean package -Dmaven.test.skip=true
```

运行编译完的 jar 包

```shell
java -jar rocketmq-dashboard-1.0.0.jar
```
