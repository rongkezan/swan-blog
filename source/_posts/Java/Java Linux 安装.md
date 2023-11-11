---
title: Java Linux 安装
date: {{ date }}
categories:
- Java
---

## Java Linux 安装

> Linux版本：CentOS 7.9

解压

```sh
tar -xvf jdk-17_linux-x64_bin.tar.gz
```

修改 `/etc/profile` 

```sh
vim /etc/profile
```

在末尾增加

```sh
export JAVA_HOME=/dist/jdk-17.0.7
export PATH=$PATH:$JAVA_HOME/bin
```

使其生效

```sh
source /etc/profile
```

查看Java版本

```sh
java --version
```

