---
title: Mysql 二进制安装(CentOS7)
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

## 修改mysql配置

修改 `/etc/my.cnf` 文件

```sh
# 修改数据目录
datadir=/usr/local/mysql/data
# 修改sock文件位置
socket=/tmp/mysql.sock
# 修改日志文件位置
log-error=/usr/local/mysql/log/mariadb.log
pid-file=/usr/local/mysql/run/mariadb.pid
```

完整配置文件 `my.cnf`

```sh
[mysqld]
datadir=/usr/local/mysql/data
socket=/tmp/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
log-error=/usr/local/mysql/log/mariadb.log
pid-file=/usr/local/mysql/run/mariadb.pid

#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
```

创建日志文件

```sh
mkdir /usr/local/mysql/log/
touch /usr/local/mysql/log/mariadb.log
mkdir /usr/local/mysql/run/
touch /usr/local/mysql/run/mariadb.pid
```

## 创建软连接

```sh
ln -s /usr/local/mysql/bin/mysql /usr/bin
```

测试是否创建成功，查看mysql版本

```sh
mysql --version
```

## 启动mysql

```sh
# 启动命令
service mysqld start
# 停止命令
service mysqld stop
# 重启命令
service mysqld restart
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
# 方式1：
mysql> ALTER USER USER() IDENTIFIED BY '123456';
```

```sh
# 方式2(如果是免密登录的话，无法使用方式1)：
update mysql.user set authentication_string=password('123456') where user='root' and Host ='localhost';
use mysql;
flush privileges;
```

重新登录mysql

```sh
mysql -uroot -p'123456'
```
