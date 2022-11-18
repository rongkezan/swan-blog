---
title: Mysql 备份
date: {{ date }}
categories:
- Mysql
---

## 备份数据

备份全部数据库

```sh
mysqldump -uroot -p'pwd' -A  > dump.sql
```

备份单个数据库

```sh
mysqldump -uroot -p'pwd' [db_name] > dump.sql
```

备份单个表

```sh
mysqldump -uroot -p'pwd' [db_name] [table] > dump.sql
```

## 还原数据

方法一

```sh
mysql> source /disk/backup/dump.sql
```

方法二

```sh
mysql -uroot -p'pwd' < /disk/backup/dump.sql
```

