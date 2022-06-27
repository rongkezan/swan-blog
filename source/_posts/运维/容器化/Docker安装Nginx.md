---
title: Docker安装Nginx
date: {{ date }}
categories:
- 运维
- 容器化
---
## 1. 复制容器内的配置文件
1.创建一个nginx容器
```powershell
docker run -d -p 80:80 --name nginx nginx:1.16.1
```
2.复制容器内的配置文件到当前目录
```powershell
docker container cp nginx:/etc/nginx .
```
3.改名并移动到`~/docker/nginx/conf`目录
```powershell
mkdir -p ~/docker/nginx
mv nginx ~/docker/nginx/conf
```
4.删除容器
```powershell
docker stop nginx
docker rm nginx
```

## 2. 创建nginx容器
```powershell
docker run -d -p 80:80 --name nginx --privileged=true \
-v ~/docker/nginx/html:/usr/share/nginx/html \
-v ~/docker/nginx/logs:/var/log/nginx \
-v ~/docker/nginx/conf:/etc/nginx \
nginx:1.16.1
```

