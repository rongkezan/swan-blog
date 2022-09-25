---
title: Kubernetes 安装
date: {{ date }}
categories:
- Kubernetes
tags:
- Kubernetes
---

## Docker

### 配置yum源

```sh
yum install -y yum-utils

yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 安装Docker

```sh
# 安装docker社区版，docker命令行，docker容器化运行环境
yum install -y docker-ce docker-ce-cli containerd.io
```

### 启动并开机启动Docker

```sh
systemctl enable docker --now
```

### 配置加速

registry-mirrors配置为自己的阿里云`容器镜像服务`的镜像加速地址

```sh
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://7yti41gr.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
systemctl daemon-reload
systemctl restart docker
```

## Kubernetes

### 安装 Kubelet

```sh
# 配置kubelet镜像源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 安装kubelet
yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes
```

### 启动并开机启动Kubelet

```sh
systemctl enable kubelet --now
```

### 安装配套组件

```sh
# 定义镜像源下载脚本
sudo tee ./images.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
done
EOF

# 设置为可执行文件且执行
chmod +x ./images.sh && ./images.sh
```

### 修改Hosts添加Master节点映射

```sh
# 所有机器添加Master的域名映射，IP地址需要修改为master机器的内网IP
echo "172.23.184.235  cluster-endpoint" >> /etc/hosts
```

### 初始化主节点

```sh
# 初始化主节点
# apiserver-advertise-address配master的ip
# serivce-cidr和pod-network-cidr不能重叠，也不能跟机器的IP重叠
kubeadm init \
--apiserver-advertise-address=172.23.184.235 \
--control-plane-endpoint=cluster-endpoint \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.168.0.0/16
```

出现以下提示表示初始化成功，根据提示依次执行

```sh
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token wzzmcd.qxn34cuou6pm1hnz \
    --discovery-token-ca-cert-hash sha256:112632af685d019af73c7b55d63f3c0d081a4249e30fe3bbf95c4d1c93e4b4bb \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token wzzmcd.qxn34cuou6pm1hnz \
    --discovery-token-ca-cert-hash sha256:112632af685d019af73c7b55d63f3c0d081a4249e30fe3bbf95c4d1c93e4b4bb 
```

如果令牌过期可以在master节点重新生成

```sh
kubeadm token create --print-join-command
```

### Node节点加入集群

加入集群

```sh
kubeadm join cluster-endpoint:6443 --token wzzmcd.qxn34cuou6pm1hnz \
    --discovery-token-ca-cert-hash sha256:112632af685d019af73c7b55d63f3c0d081a4249e30fe3bbf95c4d1c93e4b4bb 
```

配置环境变量

```sh
vim /etc/profile
# 末尾加一行
export KUBECONFIG=/etc/kubernetes/kubelet.conf
# 使配置生效
source /etc/profile
```

### 安装网格组件（Master节点）

```sh
curl https://docs.projectcalico.org/v3.18/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```

## 附录：完全卸载K8S和Docker

```sh
kubeadm reset -f
modprobe -r ipip
lsmod
rm -rf ~/.kube/
rm -rf /etc/kubernetes/
rm -rf /etc/systemd/system/kubelet.service.d
rm -rf /etc/systemd/system/kubelet.service
rm -rf /usr/bin/kube*
rm -rf /etc/cni
rm -rf /opt/cni
rm -rf /var/lib/etcd
rm -rf /var/etcd
yum remove -y kubelet kubeadm kubectl

docker stop $(docker ps -a -q)
docker rm $(docker ps -aq)
docker rmi $(docker images -q)
yum -y remove docker*

# 对于无法删除的镜像可以直接去目录删除
cd /var/lib/docker/image/overlay2/imagedb/content/sha256
rm -rf *
```

