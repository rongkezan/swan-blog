---
title: Linux 安装 minio
date: {{ date }}
categories:
- Linux
---

## 获取Minio

```sh
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
```

## 直接启动Minio

```sh
MINIO_ROOT_USER=admin MINIO_ROOT_PASSWORD=password ./minio server /mnt/data --console-address ":9001"
```

## 将Minio设置成服务启动

```sh
# 进入system目录
cd /etc/systemd/system/
# 创建 minio.service 文件
touch minio.service
```

`minio.service` 文件内容

```sh
[Unit]
Description=Minio Service
 
[Service]
Environment="MINIO_ROOT_USER=admin"
Environment="MINIO_ROOT_PASSWORD=minio123456"
ExecStart=/data/minio/minio server /data/minio/data --console-address ":9001"
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
StandardOutput=/data/minio/logs/minio.log
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
```

加载服务文件

```sh
systemctl daemon-reload
```

设计开机启动

```sh
systemctl enable minio.service
```

启动minio

```sh
systemctl start minio.service
```

查看状态

```sh
systemctl status minio.service
```

