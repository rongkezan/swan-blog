---
title: Kubernetes Dashboard
date: {{ date }}
categories:
- Kubernetes
tags:
- Kubernetes
---

### 部署 Dashboard

```sh
# 应用配置文件
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml

# 设置访问端口
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

```sh
# 找到端口在安全组放行
kubectl get svc -A |grep kubernetes-dashboard

# 出现以下信息：31719则是访问端口
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.96.121.87   <none>        8000/TCP         
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.96.3.21     <none>        443:31719/TCP   
```

访问控制台

```
https://{ip}:31719
```

### 登陆 Dashboard

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

## 