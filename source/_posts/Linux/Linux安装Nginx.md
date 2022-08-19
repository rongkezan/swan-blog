---
title: Linux安装Nginx
date: {{ date }}
categories:
- Linux
---

## Nginx安装

安装依赖包

```sh
yum -y install gcc zlib zlib-devel pcre-devel openssl openssl-devel
```

下载并解压nginx

```sh
wget http://nginx.org/download/nginx-1.18.0.tar.gz
tar -xvf nginx-1.18.0.tar.gz
```

进入nginx目录安装ssl模块

```sh
./configure --with-http_stub_status_module --with-http_ssl_module
```

执行编译

```sh
make && make install
```

nginx默认会在`/usr/local/`下创建一个`nginx`目录

启动脚本会在 `/usr/local/nginx/sbin` 目录下

启动脚本

```sh
./nginx
```

软连接启动脚本

```sh
ln -s /usr/local/nginx/sbin/nginx /usr/sbin
```

`nginx` 命令

```sh
# 启动nginx
nginx
# 关闭nginx
nginx -s stop
# 重启nginx
nginx -s reload
```

## Nginx配置

配置文件位置 `/usr/local/nginx/conf/nginx.conf`

