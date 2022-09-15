---
title: Kubernetes操作、
date: {{ date }}
categories:
- Docker
tags:
- Kubernetes
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

## 工作负载

![在这里插入图片描述](https://img-blog.csdnimg.cn/067d736577e54325ab31581b0e7665ee.png)

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

> 扩缩容、自愈、滚动更新

#### 基本操作

```sh
# 创建一个部署
kubectl create deployment mytomcat --image=tomcat:8.5.68
# 创建一个部署 -- 多副本
kubectl create deployment mytomcat --image=tomcat:8.5.68 --replicas=3
# 当删除该部署创建的Pod的时候，会自动重启一个Pod（自愈能力）
kubectl delete pod mytomcat-5c9c88c545-vhrfb
# 删除部署
kubectl delete deploy mytomcat
# 扩缩容
kubectl scale --replicas=5 deployment/my-dep
# 滚动更新
kubectl set image deployment/my-dep nginx=nginx:1.16.1
kubectl rollout status deployment/my-dep
## 版本回退
# 历史记录
kubectl rollout history deployment/my-dep
# 查看某个历史详情
kubectl rollout history deployment/my-dep --revision=2
# 回滚(回到上次)
kubectl rollout undo deployment/my-dep
# 回滚(回到指定版本)
kubectl rollout undo deployment/my-dep --to-revision=2
```

#### 使用配置文件

`test-deployment.yaml` 内容

```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: my-dep
    name: my-dep
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: my-dep
    template:
      metadata:
        labels:
          app: my-dep
      spec:
        containers:
        - image: nginx
          name: nginx
```

应用配置文件

```sh
kubectl apply -f test-deployment.yaml
```

删除指定配置文件所创建的资源

```sh
kubectl delete -f test-deployment.yaml
```

### Service

> 将一组Pods公开为网络服务的抽象方法

#### 暴露服务

将工作负载 `my-nginx` 的80端口映射到集群内部的8000端口并暴露

```sh
kubectl expose deployment my-nginx --port=8000 --target-port=80
```

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: my-nginx
  name: my-nginx
spec:
  selector:
    app: my-nginx
  ports:
  - port: 8000
    protocol: TCP
    targetPort: 80
```

#### 集群内部访问service

```sh
kubectl get service
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          40m
my-nginx     LoadBalancer   10.96.50.125   <pending>     8000:30698/TCP   6m35s
```

通过 `[ip]:[port]` 的形式访问

```sh
curl 10.96.50.125:8000
```

通过 `[serviceName].[namespace].svc`的形式访问

```sh
curl my-nginx.default.svc:8000
```

#### ClusterIP

```sh
# 只能在集群内部访问，默认type
kubectl expose deployment my-dep --port=8000 --target-port=80 --type=ClusterIP
```

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: my-dep
  name: my-dep
spec:
  ports:
  - port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    app: my-dep
  type: ClusterIP
```

#### NodePort

```sh
# 集群外部也可以访问
kubectl expose deployment my-dep --port=8000 --target-port=80 --type=NodePort
```

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: my-dep
  name: my-dep
spec:
  ports:
  - port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    app: my-dep
  type: NodePort

```

### Ingress

> Service的统一网关入口
>
> 官网：https://kubernetes.github.io/ingress-nginx/

#### 修改k8s端口限制范围

k8s的node节点的端口默认被限制在30000-32767的范围，如果要开放其他端口需要修改

```sh
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

在spec.containers.command的最后面加上 --service-cluster-ip-range 这一行，如下内容

```yaml
- --service-node-port-range=1-65535
```

重启kubectl

```sh
systemctl daemon-reload
systemctl restart kubelet
```

#### 安装Ingress

```sh
# 可以使用下面的deploy.yaml
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/baremetal/deploy.yaml
# 修改镜像
vi deploy.yaml
# 将image的值改为如下值：
registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/ingress-nginx-controller:v0.46.0
# 检查安装的结果
kubectl get pod,svc -n ingress-nginx
```

`ingress-deploy.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx

---
# Source: ingress-nginx/templates/controller-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: ingress-nginx
automountServiceAccountToken: true
---
# Source: ingress-nginx/templates/controller-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
---
# Source: ingress-nginx/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
  name: ingress-nginx
rules:
  - apiGroups:
      - ''
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ''
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
---
# Source: ingress-nginx/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
  name: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:
  - kind: ServiceAccount
    name: ingress-nginx
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/controller-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: ingress-nginx
rules:
  - apiGroups:
      - ''
    resources:
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ''
    resources:
      - configmaps
      - pods
      - secrets
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - configmaps
    resourceNames:
      - ingress-controller-leader-nginx
    verbs:
      - get
      - update
  - apiGroups:
      - ''
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create
      - patch
---
# Source: ingress-nginx/templates/controller-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:
  - kind: ServiceAccount
    name: ingress-nginx
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/controller-service-webhook.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller-admission
  namespace: ingress-nginx
spec:
  type: ClusterIP
  ports:
    - name: https-webhook
      port: 443
      targetPort: webhook
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
---
# Source: ingress-nginx/templates/controller-service.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
      nodePort: 80
    - name: https
      port: 443
      protocol: TCP
      targetPort: 443
      nodePort: 443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
---
# Source: ingress-nginx/templates/controller-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller
  revisionHistoryLimit: 10
  minReadySeconds: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/component: controller
    spec:
      dnsPolicy: ClusterFirst
      containers:
        - name: controller
          image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/ingress-nginx-controller:v0.46.0
          imagePullPolicy: IfNotPresent
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
          args:
            - /nginx-ingress-controller
            - --election-id=ingress-controller-leader
            - --ingress-class=nginx
            - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
            - --validating-webhook=:8443
            - --validating-webhook-certificate=/usr/local/certificates/cert
            - --validating-webhook-key=/usr/local/certificates/key
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            runAsUser: 101
            allowPrivilegeEscalation: true
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: LD_PRELOAD
              value: /usr/local/lib/libmimalloc.so
          livenessProbe:
            failureThreshold: 5
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
            - name: webhook
              containerPort: 8443
              protocol: TCP
          volumeMounts:
            - name: webhook-cert
              mountPath: /usr/local/certificates/
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 90Mi
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
        - name: webhook-cert
          secret:
            secretName: ingress-nginx-admission
---
# Source: ingress-nginx/templates/admission-webhooks/validating-webhook.yaml
# before changing this value, check the required kubernetes version
# https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#prerequisites
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  name: ingress-nginx-admission
webhooks:
  - name: validate.nginx.ingress.kubernetes.io
    matchPolicy: Equivalent
    rules:
      - apiGroups:
          - networking.k8s.io
        apiVersions:
          - v1beta1
        operations:
          - CREATE
          - UPDATE
        resources:
          - ingresses
    failurePolicy: Fail
    sideEffects: None
    admissionReviewVersions:
      - v1
      - v1beta1
    clientConfig:
      service:
        namespace: ingress-nginx
        name: ingress-nginx-controller-admission
        path: /networking/v1beta1/ingresses
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
rules:
  - apiGroups:
      - admissionregistration.k8s.io
    resources:
      - validatingwebhookconfigurations
    verbs:
      - get
      - update
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx-admission
subjects:
  - kind: ServiceAccount
    name: ingress-nginx-admission
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
rules:
  - apiGroups:
      - ''
    resources:
      - secrets
    verbs:
      - get
      - create
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-admission
subjects:
  - kind: ServiceAccount
    name: ingress-nginx-admission
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ingress-nginx-admission-create
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
spec:
  template:
    metadata:
      name: ingress-nginx-admission-create
      labels:
        helm.sh/chart: ingress-nginx-3.33.0
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/version: 0.47.0
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: admission-webhook
    spec:
      containers:
        - name: create
          image: docker.io/jettech/kube-webhook-certgen:v1.5.1
          imagePullPolicy: IfNotPresent
          args:
            - create
            - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc
            - --namespace=$(POD_NAMESPACE)
            - --secret-name=ingress-nginx-admission
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
      restartPolicy: OnFailure
      serviceAccountName: ingress-nginx-admission
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/job-patchWebhook.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ingress-nginx-admission-patch
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-3.33.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.47.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
spec:
  template:
    metadata:
      name: ingress-nginx-admission-patch
      labels:
        helm.sh/chart: ingress-nginx-3.33.0
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/version: 0.47.0
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: admission-webhook
    spec:
      containers:
        - name: patch
          image: docker.io/jettech/kube-webhook-certgen:v1.5.1
          imagePullPolicy: IfNotPresent
          args:
            - patch
            - --webhook-name=ingress-nginx-admission
            - --namespace=$(POD_NAMESPACE)
            - --patch-mutating=false
            - --secret-name=ingress-nginx-admission
            - --patch-failure-policy=Fail
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
      restartPolicy: OnFailure
      serviceAccountName: ingress-nginx-admission
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
```

安装完成后k8s会新建service

如图：一个是映射为80端口的32023，一个是映射443端口的30771

```sh
[root@master ~]# kubectl get service -A
NAMESPACE              NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
default                hello-server                         ClusterIP   10.96.174.151   <none>        8000/TCP                     166m
default                kubernetes                           ClusterIP   10.96.0.1       <none>        443/TCP                      4h18m
default                nginx-demo                           ClusterIP   10.96.70.26     <none>        8000/TCP                     166m
ingress-nginx          ingress-nginx-controller             NodePort    10.96.101.44    <none>        80:32023/TCP,443:30771/TCP   177m
ingress-nginx          ingress-nginx-controller-admission   ClusterIP   10.96.218.95    <none>        443/TCP                      177m
kube-system            kube-dns                             ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP       4h18m
kubernetes-dashboard   dashboard-metrics-scraper            ClusterIP   10.96.181.48    <none>        8000/TCP                     4h5m
kubernetes-dashboard   kubernetes-dashboard                 NodePort    10.96.95.53     <none>        443:31476/TCP                4h5m

```

#### 使用

应用如下配置，准备好测试环境

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-server
  template:
    metadata:
      labels:
        app: hello-server
    spec:
      containers:
      - name: hello-server
        image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/hello-server
        ports:
        - containerPort: 9000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-demo
  name: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - image: nginx
        name: nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-demo
  name: nginx-demo
spec:
  selector:
    app: nginx-demo
  ports:
  - port: 8000
    protocol: TCP
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hello-server
  name: hello-server
spec:
  selector:
    app: hello-server
  ports:
  - port: 8000
    protocol: TCP
    targetPort: 9000
```

##### 域名访问

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress  
metadata:
  name: ingress-host-bar
spec:
  ingressClassName: nginx
  rules:
  - host: "hello.atguigu.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: hello-server
            port:
              number: 8000
  - host: "demo.atguigu.com"
    http:
      paths:
      - pathType: Prefix
        path: "/nginx"
        backend:
          service:
            name: nginx-demo
            port:
              number: 8000
```

##### 路径重写

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress  
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: ingress-host-bar
spec:
  ingressClassName: nginx
  rules:
  - host: "hello.atguigu.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: hello-server
            port:
              number: 8000
  - host: "demo.atguigu.com"
    http:
      paths:
      - pathType: Prefix
        path: "/nginx(/|$)(.*)"  # 把请求会转给下面的服务，下面的服务一定要能处理这个路径，不能处理就是404
        backend:
          service:
            name: nginx-demo  ## java，比如使用路径重写，去掉前缀nginx
            port:
              number: 8000
```

##### 流量限制

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-limit-rate
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "1"
spec:
  ingressClassName: nginx
  rules:
  - host: "haha.atguigu.com"
    http:
      paths:
      - pathType: Exact
        path: "/"
        backend:
          service:
            name: nginx-demo
            port:
              number: 8000
```

##### 配置SSL

下载阿里云SSL证书，解压到服务器，执行下列命令

```sh
kubectl create secret tls app-nft-card-com-secret --key ./7995548__51nftcard.com.key --cert ./7995548__51nftcard.com.pem
```

修改`ingress-rule.yaml`增加ssl配置 

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-host-bar
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: 'true' #http 自动转https
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600" #修改代理超时时间，默认是60s
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - '51nftcard.com'
    secretName: app-nft-card-com-secret
  rules:
  - host: "hello.51nftcard.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: hello-server
            port:
              number: 8000
  - host: "demo.51nftcard.com"
    http:
      paths:
      - pathType: Prefix
        path: "/nginx(/|$)(.*)"  # 把请求会转给下面的服务，下面的服务一定要能处理这个路径，不能处理就是404
        backend:
          service:
            name: nginx-demo  ## java，比如使用路径重写，去掉前缀nginx
            port:
              number: 8000
```

应用 `ingress-rule.yaml`

```sh
kubectl apply -f ingress-rule.yaml
```

### Watch

```sh
# 每隔1秒运行一次获取Pod命令
watch -n 1 kubectl get pod
```

### 存储抽象

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



## Kubernetes部署Java微服务

### 打包

> maven打成可执行jar，上传给master服务器

### 制作镜像

> docker根据Dockerfile把包打成指定的镜像

`Dockerfile`

```dockerfile
# 拉取JDK镜像
FROM openjdk:11-jdk
MAINTAINER KEITH

# 修改服务器时间为东8区
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone

# 复制Jar包到容器内
COPY target/*.jar /root/app.jar

# 暴露8080端口
EXPOSE 8080

# 指定参数运行容器
ARG ENV
ENV JAVA_OPTS="--server.port=8080 --spring.profiles.active=${ENV}"
ENTRYPOINT ["/bin/sh","-c","java -Dfile.encoding=utf8 -Djava.security.egd=file:/dev/./urandom -jar /root/app.jar ${JAVA_OPTS}"]
```

构建Docker镜像

```sh
cd [project_dir]
docker build -t [project_name]:[version] -f Dockerfile .
```

### 推送镜像

将镜像推送给Docker Hub（阿里云镜像仓库）

```sh
# 登录阿里云Docker仓库
docker login --username=[username] registry.cn-hangzhou.aliyuncs.com
# 给镜像打标签
docker tag [image_id] registry.cn-hangzhou.aliyuncs.com/[namespace]/kube-apiserver:[image_version]
# 推送镜像
docker push registry.cn-hangzhou.aliyuncs.com/[namespace]/kube-apiserver:[image_version]
```

### 滚动更新K8S镜像

```sh
kubectl set image deployment/[deploy_name] [deploy_name]= registry.cn-hangzhou.aliyuncs.com/[namespace]/kube-apiserver:[image_version]
```

### 解决阿里云ECS搭建K8S无法拉取到阿里云镜像的问题

```sh
# 复制阿里云登录信息到kubelet目录
cp /root/.docker/config.json /var/lib/kubelet
# 重启kubelet
systemctl restart kubelet.service
```

原因：kubelet没有读取到/root/.docker/config.json的认证密钥

### 部署脚本

#### 拉取且部署

```sh
git pull
# mvn package
mvn clean package -DskipTests
# docker build
docker build -t ${IMAGE_NAME} -f Dockerfile . --build-arg ENV=$ENV
# docker tag
docker tag ${IMAGE_NAME} registry-vpc.cn-hangzhou.aliyuncs.com/nft_card/${IMAGE_NAME}
# delete image
docker rmi -f ${IMAGE_NAME}
# docker push
docker push registry-vpc.cn-hangzhou.aliyuncs.com/nft_card/${IMAGE_NAME}
# get aliyun image
ALIYUN_IMAGE=registry-vpc.cn-hangzhou.aliyuncs.com/nft_card/${IMAGE_NAME}
# update k8s deploy
if [ "$ENV"x = "prod"x ];then
  kubectl set image deployment/nft-card nft-card=${ALIYUN_IMAGE} -n prod
else
  kubectl set image deployment/nft-card nft-card=${ALIYUN_IMAGE} -n dev
fi
# 删除旧版镜像，只保留最新1个
docker images | grep nft-card | awk '{print $3}' | awk 'BEGIN{FS=" "} NR>1 {print $NF}' | xargs docker rmi
# echo
echo ${ALIYUN_IMAGE}
```

#### 仅部署

```sh
#!/bin/bash

# 部署测试环境
# ./deploy.sh nft-card dev
# 部署生产环境
# ./deploy.sh nft-card prod

if [ -z $1 ] || [ -z $2 ];then
  echo 缺少必填参数
  exit 1
fi

PROJECT_NAME=$1
ENV=$2

VERSION_FILE=/root/k8s/version/${PROJECT_NAME}.version
# 文件不存在则创建
if [ -f ${VERSION_FILE} ];then
  echo 0 > ${VERSION_FILE}
fi

# 存放自增变量
if [ -e ${VERSION_FILE} ];then
  sum=`cat ${VERSION_FILE}`
fi
offsets=`expr $sum \* 10`
sum=`expr $sum + 1`
echo $sum > ${VERSION_FILE}

# Define Value
IMAGE_NAME=${PROJECT_NAME}:2.$sum
PROJECT_DIR=/root/docker/jenkins/workspace/${PROJECT_NAME}
# Go to project dir
cd ${PROJECT_DIR}
# docker build
docker build -t ${IMAGE_NAME} -f Dockerfile . --build-arg ENV=$ENV
# docker tag
docker tag ${IMAGE_NAME} registry-vpc.cn-hangzhou.aliyuncs.com/nft_card/${IMAGE_NAME}
# delete image
docker rmi -f ${IMAGE_NAME}
# docker push
docker push registry-vpc.cn-hangzhou.aliyuncs.com/nft_card/${IMAGE_NAME}
# get aliyun image
ALIYUN_IMAGE=registry-vpc.cn-hangzhou.aliyuncs.com/nft_card/${IMAGE_NAME}
# update k8s deploy
if [ "$ENV"x = "prod"x ];then
  kubectl set image deployment/${PROJECT_NAME} ${PROJECT_NAME}=${ALIYUN_IMAGE} -n prod
else
  kubectl set image deployment/${PROJECT_NAME} ${PROJECT_NAME}=${ALIYUN_IMAGE} -n dev
fi
# echo
echo ${ALIYUN_IMAGE}
```

## 参考文献

https://www.yuque.com/leifengyang/oncloud/ctiwgo
