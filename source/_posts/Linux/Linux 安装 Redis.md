---
title: Linux 安装 Redis
date: {{ date }}
categories:
- Linux
---

## 二进制安装

```bash
yum install -y gcc
cd redis-7.0.11
make && make install
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

## 启动

指定配置文件启动

```sh
redis-server redis.conf
```

