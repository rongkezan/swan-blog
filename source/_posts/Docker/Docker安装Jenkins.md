---
title: Docker安装Jenkins
date: {{ date }}
categories:
- Docker
---

> 服务器域名以 192.168.25.100 为例

## 安装 Jenkins 镜像

**拉取镜像 jenkinsci/blueocean**

官方推荐使用的镜像是jenkinsci/blueocean，该镜像包含当前的长期支持 (LTS) 的 Jenkins 版本 (可以生产使用) ，并捆绑了所有 Blue Ocean 插件和功能。

```shell
docker pull jenkinsci/blueocean
```

**修改目录权限**

因为当映射本地数据卷时，/docker/jenkins目录的拥有者为root用户，而容器中jenkins user的uid为1000

```shell
chown -R 1000:1000 /docker/jenkins
```
**运行 Jenkins 容器**

```shell
docker run \
  --name jenkins \
  -d \
  -p 8000:8080 \
  -p 50000:50000 \
  -v /docker/jenkins:/var/jenkins_home \
  -e JAVA_OPTS=-Duser.timezone=Asia/Shanghai \
  jenkinsci/blueocean
```
## 配置 Jenkins 安装插件国内地址

**在 jenkins 安装目录下找 default.json**

```shell
find . -name default.json
```

**替换 default.json 中的内容**

```shell
sed -i 's/www.google.com/www.baidu.com/g' default.json
sed -i 's/updates.jenkins-ci.org\/download/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json
```

**重启 jenkins**

```shell
http://192.168.25.100:8000/restart
```

## 登陆 Jenkins

**访问 Jenkins 页面**

```shell
http://192.168.25.100:8000
```

**输入管理员密码**

获取 Docker 映射地址的密码

```shell
cat /docker/jenkins/secrets/initialAdminPassword
85770376692448b7b6a8e301fb437848
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210305234455971.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 配置 maven jdk git

**安装插件 Maven Integration，重启 Jenkins**

Manage Jenkins -> Manage Plugins -> Maven Intergration -> Download now and install after restart

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210307122414262.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

**进入容器查询 jdk 和 git 的路径**

```shell
# 进入容器
docker exec -it jenkins bash
# 获取jdk的路径
echo $JAVA_HOME
# 获取git的路径
which git
```
**配置路径**

Manage Jenkins -> Global Tool Configuration -> Update -> Save

![JDK](https://img-blog.csdnimg.cn/20210307124810429.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 构建项目

New Item -> 构建一个maven项目
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210307131950770.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

配置 Git 仓库地址

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210307132120544.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

Build Now

构建完成后，源码路径：/docker/jenkins/workspace/Test

## 配置 Gitee

**安装 Gitee 插件**

Manage Jenkins -> Plugin Manager -> Search gitee -> Download now and install after restart

**配置 Gitee**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210307155434803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

**配置构建任务**

General -> gitee链接

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210307161531768.png)

Source Code Management -> 选择 Git -> 配置 Repository URL 和 Credentials

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210307161629421.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)