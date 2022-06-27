---
title: Docker自定义镜像
date: {{ date }}
categories:
- 运维
- 容器化
---

构建Docker镜像

```sh
# 如果你的构建文件名叫Dockerfile，那么'-f Dockerfile'可以省略
docker build -t java-demo:v1.0 -f Dockerfile .  
```

启动Docker镜像

```sh
docker run -d -p 14251:14251 --name my-java java-demo:v1.0
```

分享镜像

```sh
docker login
# 前面的名字需要跟你仓库名对应
docker tag java-demo:v1.0 rongkezan/java-demo:v1.0
docker push rongkezan/java-demo:v1.0
```

