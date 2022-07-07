---
title: Kubernetes
date: {{ date }}
categories:
- 运维
- 容器化
---
## Kubernetes 特性

- **服务发现和负载均衡**
  Kubernetes 可以使用 DNS 名称或自己的 IP 地址公开容器，如果进入容器的流量很大， Kubernetes 可以负载均衡并分配网络流量，从而使部署稳定。
- **存储编排**
  Kubernetes 允许你自动挂载你选择的存储系统，例如本地存储、公共云提供商等。
- **自动部署和回滚**
  你可以使用 Kubernetes 描述已部署容器的所需状态，它可以以受控的速率将实际状态 更改为期望状态。例如，你可以自动化 Kubernetes 来为你的部署创建新容器， 删除现有容器并将它们的所有资源用于新容器。
- **自动完成装箱计算**
  Kubernetes 允许你指定每个容器所需 CPU 和内存（RAM）。 当容器指定了资源请求时，Kubernetes 可以做出更好的决策来管理容器的资源。
- **自我修复**
  Kubernetes 重新启动失败的容器、替换容器、杀死不响应用户定义的 运行状况检查的容器，并且在准备好服务之前不将其通告给客户端。
- **密钥与配置管理**
  Kubernetes 允许你存储和管理敏感信息，例如密码、OAuth 令牌和 ssh 密钥。 你可以在不重建容器镜像的情况下部署和更新密钥和应用程序配置，也无需在堆栈配置中暴露密钥。

## 工作方式

Kubernetes Cluster = N Master Node + N Worker Node (N >= 1)

## Mac安装Kubernetes

### 1. 启动Kubernetes

安装好Docker For Mac之后，会携带Kubernetes

在Preferences中：

1. 修改Docker Engine

   ```json
     "registry-mirrors": [
       "https://docker.mirrors.ustc.edu.cn",
       "https://cr.console.aliyun.com/"
     ]
   ```

2. 启动Kubernetes

   Kubernetes -> Enable Kubernetes -> Apply & Restart

### 2. 验证是否启动成功

```sh
kubectl cluster-info
kubectl get nodes
kubectl describe node
```

### 3. 部署Dashboard

部署Dashboard

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

启动Dashboard

```sh
kubectl proxy
```

访问控制台

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

### 4. 登陆Dashboard

1. 新建文件`dashboard-adminuser.yaml`

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
kubectl apply -f dashboard-adminuser.yaml
```

3. 生成登陆的Token

```sh
kubectl -n kubernetes-dashboard create token admin-user
```

4. 得到如下格式的Token

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImVsb2s5NlluYWtYMDJNWmVPZFBZNmo5MU11el82VGZwR2gtVUE0OTlhY1UifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjU0Njg2MzM1LCJpYXQiOjE2NTQ2ODI3MzUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJhZG1pbi11c2VyIiwidWlkIjoiYWExY2Y0MTAtODdlNS00MDY2LWI4ODgtMTg1MmE4NTg5NzFjIn19LCJuYmYiOjE2NTQ2ODI3MzUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbi11c2VyIn0.UCSlwuNnIVQnsopHXG8PvLMDmcbtig3S8TCtN716xD4yITdpc4RqwmxzKJx1QI2L2bHKEnwMD38u96Q4qQnGWzcJ1R8MDtZc7gGtFWk51bgoie6lOtTyf4n1WSkSBzd2UFppMfDB6jiZ2vDR02crhazFgOO0dVtRGUfclDLOm2x1pTemdZJDQcR9TywKDXyQeLvGJXEi_N_a43AhnZmbI_OcadiRCc-hCwlWXWLEZSf0405t_nrXwA8NwzWjT1qJ8gDsAWHDoifS_2xoAETeP0ubhRK9HmrYa20aMHRc_M1HBiwbossNNET_iKdUqLYUNrac2r5SYQ72ia_qG4inRA
```

5. 使用Token登陆Dashboard

![在这里插入图片描述](https://img-blog.csdnimg.cn/a4010b6b0b024cf59734eaded23817b5.png)

6. 测试完成删除用户

```sh
kubectl -n kubernetes-dashboard delete serviceaccount admin-user
```

## Kubernetes操作

### Namespace

#### 基本操作

```sh
# 获取命名空间
kubectl get ns
# 创建命名空间
kubectl create ns test-ns
# 删除命名空间
kubectl delete ns test-ns
```

#### 使用配置文件操作

```sh
vi test-ns.yaml
```

`test-ns.yaml` 内容

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-ns
```

应用配置文件

```sh
kubectl apply -f test-ns.yaml
```

删除指定配置文件所创建的资源

```sh
kubectl delete -f test-ns.yaml
```

### Pod

> 运行中的一组容器，Pod是Kubernetes中应用的最小单位

#### 基本操作

```sh
# 启动一个nginx镜像的Pod
kubectl run mynginx --image=nginx
# 获取名为mynginx的Pod的描述
kubectl describe pod mynginx
# 删除Pod
kubectl delete pod mynginx
# 查看Pod的运行日志
kubectl logs mynginx
# 打印Pod的更完善的信息
kubectl get pod -owide
# 进入容器
kubectl exec -it mynginx -- /bin/bash
```

#### 使用配置文件操作

创建配置文件

```sh
vi test-pod.yaml
```

`test-pod.yaml` 内容

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mynginx
  name: mynginx
spec:
  containers:
    - image: nginx
      name: mynginx
```

应用配置文件

```sh
kubectl apply -f test-pod.yaml
```

删除指定配置文件所创建的资源

```sh
kubectl delete -f test-pod.yaml
```

### Deployment

```sh
# 创建一个部署
kubectl create deployment mytomcat --image=tomcat:8.5.68
# 创建一个部署 -- 多副本
kubectl create deployment mytomcat --image=tomcat:8.5.68 --replicas=3
# 当删除该部署创建的Pod的时候，会自动重启一个Pod（自愈能力）
kubectl delete pod mytomcat-5c9c88c545-vhrfb
# 删除部署
kubectl delete deploy mytomcat
```

### Watch

```sh
# 每隔1秒运行一次获取Pod命令
watch -n 1 kubectl get pod
```

