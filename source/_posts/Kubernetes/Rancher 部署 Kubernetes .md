---
title: Rancher 部署 Kubernetes
date: {{ date }}
categories:
- Kubernetes
tags:
- Kubernetes
- Rancher
---

## Docker 安装

参考 [Docker 安装](https://rongkezan.github.io/swan-blog/2022/11/01/Docker/Docker%20%E5%AE%89%E8%A3%85/)

## Rancher 安装

```sh
docker pull rancher/rancher:v2.5.15
```

```sh
mkdir -p /opt/data/rancher_data
```

```sh
docker run -d --privileged \
-p 80:80 -p 443:443 \
-v /opt/data/rancher_data:/var/lib/rancher \
--restart=always \
--name rancher-2.5.15 \
rancher/rancher:v2.5.15
```

## Rancher 使用

### Rancher 控制台

设置密码，URL后登录到控制台，将语言改为中文

![在这里插入图片描述](https://img-blog.csdnimg.cn/e1d0dba3a80c4cfd988e1c25e17d9bd6.png)

### 通过 Rancher 创建 K8S 集群

添加集群 -> 自定义集群

![在这里插入图片描述](https://img-blog.csdnimg.cn/889961c3c0274e6297b829070b45d8f1.png)

![image-20221108174742476](C:\Users\76405\AppData\Roaming\Typora\typora-user-images\image-20221108174742476.png)

输入集群名称，其他配置默认，点击页面最下方的 `下一步 ` 完成集群创建

![在这里插入图片描述](https://img-blog.csdnimg.cn/56936e1ffe3b4f5b950b47d284110d8c.png)

### 添加 Master、Worker 节点

![在这里插入图片描述](https://img-blog.csdnimg.cn/79925989196040d5a8830fabe07ae1c5.png)

#### 添加 Master 节点

```sh
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.5.15 --server https://172.23.184.237 --token bqlz9dfgp2rgrw9h9n96mmf8n5r5f5746qmtlqhv4mrm29nmkpfffk --ca-checksum ae74148c06e6b6a28df42fa95e691033adc54b4c272538749861246ce0aaec99 --etcd --controlplane
```

#### 添加 Worker 节点

```sh
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.5.15 --server https://172.23.184.237 --token bqlz9dfgp2rgrw9h9n96mmf8n5r5f5746qmtlqhv4mrm29nmkpfffk --ca-checksum ae74148c06e6b6a28df42fa95e691033adc54b4c272538749861246ce0aaec99 --worker
```

#### 添加成功

添加成功后如下图所示

![在这里插入图片描述](https://img-blog.csdnimg.cn/8ca4ae2c7e574a90be1aeef937220557.png)

### 通过 Rancher 执行 K8S 命令

![在这里插入图片描述](https://img-blog.csdnimg.cn/04cf0b6731ca43e7aaf9f27acbe41e5d.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/8a7a718a05aa4bbdb7fae9c25979a4c9.png)