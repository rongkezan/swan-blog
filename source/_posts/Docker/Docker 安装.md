---
title: Docker 安装
date: {{ date }}
categories:
- Docker
---

## Docker版本

Docker 分为 CE 和 EE 两大版本。

CE 即社区版（免费，支持周期 7 个月）

EE 即企业版，强调安全，付费使用，支持周期 24 个月

Docker CE 分为 stable test 和 nightly 三个更新频道。

官方网站上有各种环境下的安装指南，这里主要介绍Docker CE 在 CentOS上的安装。

## 卸载旧版本的Docker

```sh
yum remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine \
                  docker-ce
```

## Docker 安装

### 安装 yum-utils

```sh
yum install -y yum-utils
```

### 设置镜像源

```sh
yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 安装 Docker

```sh
yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 启动并开机运行 Docker

```sh
systemctl enable docker --now
```

### 查看 Docker 版本

```sh
docker -v
```

## Docker 配置

### 配置镜像加速

```sh
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://7yti41gr.mirror.aliyuncs.com"]
}
EOF
```

重新加载配置文件，重启docker使其生效

```sh
systemctl daemon-reload
systemctl restart docker
```

