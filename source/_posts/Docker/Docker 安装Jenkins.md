---
title: Docker 安装Jenkins
date: {{ date }}
categories:
- Docker
---

> 服务器域名以 192.168.25.100 为例

## Jenkins 安装

### 拉取镜像

```shell
docker pull jenkins/jenkins:2.366-jdk11
```

### 创建目录

```sh
mkdir -p /root/docker/jenkins
```

### 修改目录权限

> 将maven和jdk下载好后放入 `/root/docker/`目录下

因为当映射本地数据卷时，`/root/docker/jenkins`目录的拥有者为root用户而容器中`jenkins user`的`uid`为1000

```shell
chown -R 1000:1000 /root/docker/jenkins
chown -R 1000:1000 /root/docker/maven
chown -R 1000:1000 /root/docker/jdk11
```
### 运行 Jenkins 容器

```sh
docker run -d \
    -p 8000:8080 \
    -p 50000:50000 \
    -v /root/docker/jenkins:/var/jenkins_home \
    -v /root/docker/maven:/var/maven \
    -v /root/docker/jdk11:/var/jdk11 \
    -v /etc/localtime:/etc/localtime \
    -v /proc:/host/proc \
    --restart=always \
    --name=jenkins \
    --privileged=true \
    --user=root \
    jenkins/jenkins:2.366-jdk11
```

1. 映射端口8080到8000，50000到50000

2. 挂载 `jenkins_home` 到宿主机

3. 挂载 `maven` 目录

4. 挂载 `jdk11` 目录

5. 同步宿主机时间

6. 容器内部操作宿主机shell指令

   ```sh
   # docker里面执行
   nsenter --mount=/host/proc/1/ns/mnt sh -c "ls /root"
   ```

   

### 配置 Jenkins 插件镜像源

修改 `/var/jenkins_home/updates/default.json`文件里的镜像源地址

```sh
sed -i 's#https://updates.jenkins.io/download#https://mirrors.tuna.tsinghua.edu.cn/jenkins#g' default.json
sed -i 's#https://www.google.com#https://www.baidu.com#g' default.json
```

将 `default.json` 上传到服务器，使用nginx指向到该文件 `https://static.51nftcard.com/update-center.json`

修改`/var/jenkins_home/hudson.model.UpdateCenter.xml`文件里的镜像源地址

```xml
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://static.51nftcard.com/update-center.json</url>
  </site>
</sites>
```

重启 `jenkins`

```sh
docker restart jenkins
```

## Jenkins使用

### 登录 Jenkins

访问 Jenkins 页面

```shell
http://192.168.25.100:8000
```

获取管理员密码

```shell
cat /root/docker/jenkins/secrets/initialAdminPassword
```

```sh
85770376692448b7b6a8e301fb437848
```

登录后安装推荐插件

### 配置 maven jdk git

#### 安装插件

> Manage Jenkins -> Manage Plugins

1. Maven Integration
2. Git Parameter

#### 全局配置

> Manage Jenkins -> Global Tool Configuration

![在这里插入图片描述](https://img-blog.csdnimg.cn/f71852ddbfe040eca8229800b13e7858.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/b5e8acc99fbe40498db0d20d7cf305f2.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/3945462ac7de47ca90187876ee6ae8ec.png)

### 构建Maven项目

#### 新建项目

New Item -> Maven Project

#### 配置Git参数化构建

This project is parameterized -> Add Parameter : Git Parameter

![在这里插入图片描述](https://img-blog.csdnimg.cn/7a07e7b5bc124a44b987141efead36a0.png)

#### 配置Git仓库地址

Source Code Management

![在这里插入图片描述](https://img-blog.csdnimg.cn/c14d1a1e276547588ca128c7719ffc3d.png)

#### 配置Maven脚本

![在这里插入图片描述](https://img-blog.csdnimg.cn/f83b5eb8745e469f81c5a9228538ee5c.png)

## 执行Shell脚本

```sh
nsenter --mount=/host/proc/1/ns/mnt sh -c "ls /root"
```



