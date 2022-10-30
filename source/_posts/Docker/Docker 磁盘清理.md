---
title: Docker 磁盘清理
date: {{ date }}
categories:
- Docker
---

## 查看Linux磁盘使用情况

```sh
df -h
```

```sh
du -h --max-depth=1
```

## 查看Docker文件夹磁盘使用情况

```sh
du -hs /var/lib/docker/
```

```sh
# 表示Docker文件夹使用了1.8G
1.8G	/var/lib/docker/
```

## 查看Docker磁盘使用情况（类似于df）

```sh
docker system df
```

```sh
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          6         6         632.1MB   3.552MB (0%)
Containers      10        10        2.294kB   0B (0%)
Local Volumes   1         0         517.5MB   517.5MB (100%)
Build Cache     0         0         0B        0B
```
