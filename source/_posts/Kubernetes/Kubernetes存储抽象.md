---
title: Kubernetes存储抽象
date: {{ date }}
categories:
- Kubernetes
tags:
- Kubernetes
---

### NFS

#### 所有节点安装nfs

```sh
yum install -y nfs-utils
```

#### 主节点配置

```sh
# nfs主节点
echo "/nfs/data/ *(insecure,rw,sync,no_root_squash)" > /etc/exports

mkdir -p /nfs/data
systemctl enable rpcbind --now
systemctl enable nfs-server --now
# 配置生效
exportfs -r
# 查看nfs配置
exportfs
```

#### 从节点配置

```sh
# 检查主节点有哪些目录可以挂载
showmount -e 172.23.184.235
# 挂载数据目录
mkdir -p /nfs/data
mount -t nfs 172.23.184.235:/nfs/data /nfs/data
# 写入一个测试文件
echo "hello nfs server" > /nfs/data/test.txt
```

#### 原生方式挂载

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-pv-demo
  name: nginx-pv-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-nfs-demo
  template:
    metadata:
      labels:
        app: nginx-nfs-demo
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          nfs:
            server: 172.31.0.4
            path: /nfs/data/nginx-pv
```

### 