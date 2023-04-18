---
title: Linux 安装 Redis
date: {{ date }}
categories:
- Linux
---

## 安装

```sh
# 解压
tar -zxvf redis-7.0.5.tar.gz
# 执行编译安装
cd redis-7.0.5
make && make install
# 默认安装路径在 /usr/local/bin 下
cd /usr/local/bin
```

## 启动命令

```sh
# redis 服务端启动
redis-server
# redis 客户端启动
redis-cli
# redis 哨兵启动
redis-sentinel
```

