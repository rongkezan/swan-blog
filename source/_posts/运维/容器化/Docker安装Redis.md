---
title: Docker安装Redis
date: {{ date }}
categories:
- 运维
- 容器化
---
## 1. 创建配置文件
#### 1.1 创建目录
```java
mkdir -p /docker/redis/conf
```
#### 1.2 将配置文件复制进去
找一份配置文件将其复制进去
修改以下参数
```sh
# 将其注释
# bind 127.0.0.1

# 将其注释，因为与docker的-d冲突
# daemonize yes
```
## 2. 创建容器
```sh
docker run -d -p 6379:6379 --name redis --privileged=true \
-v /docker/redis/data:/data \
-v /docker/redis/conf/redis.conf:/etc/redis/redis.conf \
redis redis-server /etc/redis/redis.conf \
--appendonly yes
```
## 3. 运行容器中的redis客户端
```sh
docker exec -it redis redis-cli
```
查看Redis版本
```sh
docker exec -it redis redis-server -v
```
