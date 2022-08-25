---
title: Kubernetes安装
date: {{ date }}
categories:
- Docker
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

### 安装网格组件（Master节点）

```sh
curl https://docs.projectcalico.org/v3.18/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```

## Kubernetes Dashboard

### 部署

部署Dashboard

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

设置访问端口

```sh
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

> type: ClusterIP 改为 NodePort

```sh
# 找到端口在安全组放行
kubectl get svc -A |grep kubernetes-dashboard
```

出现以下信息：31719则是访问端口

```sh
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.96.121.87   <none>        8000/TCP         
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.96.3.21     <none>        443:31719/TCP   
```

访问控制台

```
https://{ip}:31719
```

### 登陆

1. 新建文件`dashboard-admin.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

2. 应用文件

```sh
kubectl apply -f dashboard-admin.yaml
```

3. 获取访问令牌

```sh
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

4. 得到如下格式的Token

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IlR2NjU1ckNvajQ5MlJhblRaZU9VdVNxXzM0V2Y4dDZWRVRCdHUzcFRETnMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWM0Zm00Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlOTcwY2M1OC1iNzllLTQ5OTEtOWZjNy05ODM1NGQzM2QyNDIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.ZIostIO5gRytlHlHflQDqETHCYg7grVUyoLhb7bFRRgkIkxFAgv0k-KbKlQmhwQDXJG4nqi7h60tVtao9vGZtkwV5vazJvBG38XeK1P_ThYI33hZ70Mna-aCCca6dzjqqFg2bXW8Ndoc3LZT9f7hBub3cSfzwlQAUF2HFzpizytnHww2Dh5sH0T-mEA4t3qQByafR3vwaV5uEMlkx0PHgB5KKEbKWhYcGyppZTaLQ5WP65htf4j2hHfJtGA-FqVDps6Zk8JDhKaWfvlhrb4T6RGFzNcX47TBQU-OlSPFzADuo-u3N0S58my3yFZNoOVn1YMgzXNY-99YneZZqD93XA
```

5. 使用Token登陆Dashboard

![在这里插入图片描述](https://img-blog.csdnimg.cn/a4010b6b0b024cf59734eaded23817b5.png)

6. 修改token的有效时间

```sh
kubectl get deploy kubernetes-dashboard -o yaml -n kubernetes-dashboard > 88.yaml
```

```sh
vim 88.yaml
```

添加参数：`--token-ttl=43200`

```yaml
    spec:
      containers:
      - args:
        - --auto-generate-certificates
        - --namespace=kubernetes-dashboard
        - --token-ttl=43200  # 添加
```

应用配置

```sh
kubectl apply -f 88.yaml
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

