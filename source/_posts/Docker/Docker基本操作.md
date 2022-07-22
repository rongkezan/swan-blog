---
title: Docker基本操作
date: {{ date }}
categories:
- Docker
---
## 镜像操作

```sh
# 安装docker
yum -y install docker 
# 启动docker
systemctl start docker
# 停止docker
systemctl stop docker
# 设置开机启动
systemctl enable docker
# 帮助命令
docker --help
# 搜索镜像
docker search mysql
# 拉取镜像
docker pull mysql
# 可以拉取指定版本
docker pull mysql:5.5
# 查询本地镜像
docker images
# 删除本地镜像（根据 IMAGE_ID）
docker rmi 0f3e07c0138f
```

## 容器操作

```sh
# 创建并启动容器 (-d 表示后台运行)
docker run --name mytomcat -d tomcat:latest
# 查看创建了哪些容器
docker ps -a
# 查看哪些容器正在运行
docker ps
# 停止容器 （根据 容器名称 或 容器ID）
docker stop mytomcat
# 启动容器 （根据 容器名称 或 容器ID）
docker start mytomcat
# 删除容器 （根据 容器名称 或 容器ID）
docker rm mytomcat
# 配置端口映射（-p）主机端口：容器端口
docker run --name mytomcat -d -p 8888:8080 tomcat:latest
# 可以使用一个镜像启动多个容器 例如：启动3个Tomcat服务器，端口为8881,8882,8883
docker run --name mytomcat1 -d -p 8881:8080 tomcat:latest
docker run --name mytomcat2 -d -p 8882:8080 tomcat:latest
docker run --name mytomcat3 -d -p 8883:8080 tomcat:latest
# 查看容器日志
docker logs mytomcat
# 进入容器（exit 命令退出）
docker exec -it mytomcat bash 
# 镜像提交
docker commit -m="提交的描述信息" -a="作者" [容器ID] [要创建的目标镜像名]:[标签名]
```

更多命令：[https://docs.docker.com/engine/reference/commandline/docker](https://docs.docker.com/engine/reference/commandline/docker)