---
title: Docker 安装Mysql
date: {{ date }}
categories:
- Docker
---
## 1. 创建容器
```java
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=root --net=host \
-v /docker/mysql/data:/mysql/data \
-v /docker/mysql/logs/errlog:/mysql/data/discard/errlog \
-v /docker/mysql/logs/binlog:/mysql/data/discard/logdir/binlog \
-v /docker/mysql/logs/redolog:/mysql/data/discard/logdir/redolog \
-v /docker/mysql/tmpdir:/mysql/data/discard/tmpdir \
-v /docker/mysql/logs/slowlog:/mysql/data/discard/slowlog \
-v /docker/mysql:/mysql/data/discard/other \
-v /docker/mysql/conf:/etc/mysql/conf.d \
--privileged=true mysql
```
## 2. 进入容器
```java
docker exec -it mysql /bin/bash
```
## 3. 找到容器内配置文件位置
```java
mysql --help | grep my.cnf

# 按照路径优先排序，可能出现在以下路径，本人路径为 /etc/mysql/my.cnf
order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf 
```
将配置文件内容复制一份到`/docker/mysql/conf`，然后就可以根据自己的需求修改配置了。
