---
title: Mysql 压缩包安装
date: {{ date }}
categories:
- Mysql
---

## 下载官网压缩包

下载地址

https://downloads.mysql.com/archives/community/

## 下载mysql依赖

```sh
# 查询是否有libaio
rpm -qa | grep libaio
# 下载libaio
yum -y install libaio
```

## 解压并移动

```sh
tar -xzvf mysql-5.7.39-el7-x86_64.tar.gz
mv mysql-5.7.39-el7-x86_64.tar.gz /usr/local/mysql
```

## 创建mysql用户组

```sh
groupadd mysql  
useradd -g mysql mysql
chown -R mysql.mysql /usr/local/mysql/
```

## 初始化mysql

在 `bin` 目录下

```sh
./mysqld --initialize --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data
```

## 将mysql服务脚本放到系统服务中

```sh
cp -a support-files/mysql.server /etc/init.d/mysqld
```

## 修改mysql配置

修改 `/etc/my.cnf` 文件

```
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/tmp/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid

#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
```

创建日志文件

```sh
mkdir /var/log/mariadb
touch /var/log/mariadb/mariadb.log
mkdir /var/run/mariadb
touch /var/run/mariadb/mariadb.pid
```

增加日志文件权限

```sh
chown -R mysql:mysql /var/log
```

## 创建软连接

```sh
ln -s /usr/local/mysql/bin/mysql /usr/bin
```

测试是否创建成功，查看mysql版本

```sh
mysql --version
```

## 登录mysql

查看初始密码

```sh
cat /root/.mysql_secret
7IG=CSh_A&V1
```

登录mysql

```sh
mysql -uroot -p'7IG=CSh_A&V1'
```

修改密码

```sh
mysql> ALTER USER USER() IDENTIFIED BY '123456';
```

重新登录mysql

```sh
mysql -uroot -p'123456'
```

