---
title: Linux定时任务
date: {{ date }}
categories:
- Linux
---

## 新增定时任务

```sh
crontab -e
```

```sh
# 每分钟执行一次
* * * * * echo Hello World
```

```sh
-u指定一个用户

-l列出某个用户的任务计划

-r删除某个用户的任务(不添加用户即删除所有的任务)

-e编辑某个用户的任务

# 示例
crontab -u root -e
```

## 示例

```sh
# 每月1号的11点执行，删除Docker镜像
0 11 1 * * docker images | grep nft | awk '{print $3}' | awk 'BEGIN{FS=" "} NR>1 {print $NF}' | xargs docker rmi
```

