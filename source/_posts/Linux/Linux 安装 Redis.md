---
title: Linux 安装 Redis
date: {{ date }}
categories:
- Linux
---

## yum安装

```sh
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

yum -y install redis

systemctl start redis

# 配置文件路径 /etc/redis.conf
```

## 二进制安装

```bash
yum install -y gcc
cd redis-7.0.11
make && make install
```

## 启动命令

```sh
systemctl start redis
systemctl restart redis
```

## 配置

### 配置外网访问

注释bind

```properties
# bind 127.0.0.1
```

关闭保护模式

```properties
protected-mode no
```

### 配置后台启动

```sh
daemonize yes
```

### 配置密码

```sh
requirepass 123456
```
