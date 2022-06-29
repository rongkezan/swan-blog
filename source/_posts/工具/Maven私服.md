---
title: Maven私服
date: {{ date }}
categories:
- 工具
- Maven
---

## Maven配置私服

### 配置方式

Maven配置私服有两种方式

1. `settings.xml` ：全局配置方式
2. `pom.xml` ：项目独享方式

若 `settings.xml` 和 `pom.xml` 同时配置了，则以 `pom.xml` 为准

当我们在maven使用maven-public仓库地址的时候，会按照如下顺序访问：本地仓库 -> 私服仓库 -> 中央仓库

PS：私服仓库和中央仓库的顺序可以在Maven私服中配置

### Maven配置使用私服（下载依赖）

1. 通过 `settings.xml` 配置

```xml
<profiles>
  <profile>
    <id>local</id>
    <repositories>
      <repository>
        <id>maven-public</id>
        <url>http://127.0.0.1:8081/repository/maven-public/</url>
        <releases>
          <enabled>true</enabled>
        </releases>
        <snapshots>
          <enabled>true</enabled>
          <updatePolicy>always</updatePolicy>
        </snapshots>
      </repository>
    </repositories>
  </profile>
</profiles>

<activeProfiles>
  <activeProfile>local</activeProfile>
</activeProfiles>
```

如果并没有搭建私服，可以直接使用阿里云Maven仓库

```xml
<profiles>
  <profile>
    <id>local</id>
    <repositories>
      <repository>
        <id>maven-public</id>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
        <releases>
          <enabled>true</enabled>
        </releases>
        <snapshots>
          <enabled>true</enabled>
          <updatePolicy>always</updatePolicy>
        </snapshots>
      </repository>
    </repositories>
  </profile>
</profiles>

<activeProfiles>
  <activeProfile>local</activeProfile>
</activeProfiles>
```

2. 通过 `pom.xml` 配置

```xml
<repositories>
  <repository>
    <id>maven-nexus</id>
    <name>maven-nexus</name>
    <url>http://127.0.0.1:8081/repository/maven-public/</url>
    <releases>
      <enabled>true</enabled>
    </releases>
    <snapshots>
      <enabled>true</enabled>
      <updatePolicy>always</updatePolicy>
    </snapshots>
  </repository>
</repositories>
```

如果并没有搭建私服，可以直接使用阿里云Maven仓库

```xml
<repositories>
   <repository>
      <id>maven-aliyun</id>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <releases>
         <enabled>true</enabled>
      </releases>
      <snapshots>
         <enabled>true</enabled>
         <updatePolicy>always</updatePolicy>
         <checksumPolicy>fail</checksumPolicy>
      </snapshots>
   </repository>
</repositories>
```

### Maven配置使用私服（下载插件）

1. 通过 `settings.xml` 配置

```xml
<profiles>
    <profile>
        <id>local</id>
        <pluginRepositories>
            <pluginRepository>
                <id>maven-public</id>
                <url>http://127.0.0.1:8081/repository/maven-public/</url>
                <releases>
                    <enabled>true</enabled>
                </releases>
                <snapshots>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </snapshots>
            </pluginRepository>
        </pluginRepositories>
    </profile>
</profiles>

<activeProfiles>
  <activeProfile>local</activeProfile>
</activeProfiles>
```

2. 通过 `pom.xml` 配置

```xml
<pluginRepositories>
    <pluginRepository>
        <id>maven-nexus</id>
        <name>maven-nexus</name>
        <url>http://127.0.0.1:8081/nexus/repository/maven-public/</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </pluginRepository>
</pluginRepositories>
```

### Maven配置使用私服（发布依赖）

1. 修改 `settings.xml`

```xml
<servers>
    <server>
        <id>releases</id>
        <username>admin</username>
        <password>123456</password>
    </server>
    <server>
        <id>snapshots</id>
        <username>admin</username>
        <password>123456</password>
    </server>
</servers>
```

2. 修改 `pom.xml`

> 注意：repository中的id要与server中的id一致

```xml
<distributionManagement>
    <repository>
        <id>releases</id>
        <name>Releases</name>
        <url>http://127.0.0.1:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>snapshots</id>
        <name>Snapshot</name>
        <url>http://127.0.0.1:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

### settings.xml 私服的完整配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              http://maven.apache.org/xsd/settings-1.0.0.xsd">

    <localRepository>/usr/local/maven/repository</localRepository>

    <servers>
        <server>
            <id>my-releases</id>
            <username>admin</username>
            <password>123456</password>
        </server>
        <server>
            <id>my-snapshot</id>
            <username>admin</username>
            <password>123456</password>
        </server>
    </servers>

    <profiles>
        <profile>
            <id>local</id>
            <repositories>
                <repository>
                    <id>maven-public</id>
                    <url>http://127.0.0.1:8081/repository/maven-public/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <id>maven-public</id>
                    <url>http://127.0.0.1:8081/repository/maven-public/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>local</activeProfile>
    </activeProfiles>
</settings>
```

## Nexus Management Repository

### Nexus安装（Docker）

```sh
# 拉取Nexus镜像
docker pull sonatype/nexus3
# 创建宿主机目录
mkdir –vp /usr/local/nexus-data
# 运行Nexus容器
docker run -d --name nexus3 -p 8081:8081 -v /usr/local/nexus-data:/var/nexus-data sonatype/nexus3
```

浏览器访问 http://127.0.0.1:8081 进入Nexus控制台

### Nexus登陆

```sh
# 进入Nexus容器
docker exec -it 4057a8835ca8 /bin/bash
# 查看默认密码
cat nexus-data/admin.password
e806f582-5ca1-4e3f-a5dc-65044aa1cd0f
```

输入账号密码：

```
admin
e806f582-5ca1-4e3f-a5dc-65044aa1cd0f
```

登陆后修改密码即可

### Nexus仓库说明

#### 默认仓库说明

- **maven-central**：**maven** 中央库，默认从 **https://repo1.maven.org/maven2/** 拉取 **jar**
- **maven-releases**：私库发行版 **jar**，初次安装请将 **Deployment policy** 设置为 **Allow redeploy**
- **maven-snapshots**：私库快照（调试版本）**jar**
- **maven-public**：仓库分组，把上面三个仓库组合在一起对外提供服务，在本地 **maven** 基础配置 **settings.xml** 或项目 **pom.xml** 中使用

#### 仓库类型说明

- **group**：这是一个仓库聚合的概念，用户仓库地址选择 **Group** 的地址，即可访问 **Group** 中配置的，用于方便开发人员自己设定的仓库。**maven-public** 就是一个 **Group** 类型的仓库，内部设置了多个仓库，访问顺序取决于配置顺序，**3.x** 默认为 **Releases**、**Snapshots**、**Central**，当然你也可以自己设置。
- **hosted**：私有仓库，内部项目的发布仓库，专门用来存储我们自己生成的 **jar** 文件
- **snapshots**：本地项目的快照仓库
- **releases**： 本地项目发布的正式版本
- **proxy**：代理类型，从远程中央仓库中寻找数据的仓库（可以点击对应的仓库的 **Configuration** 页签下 **Remote Storage** 属性的值即被代理的远程仓库的路径），如可配置阿里云 **maven** 仓库
- **central**：中央仓库

### Nexus仓库操作

maven-central 中可以修改中央仓库地址为阿里云镜像地址 http://maven.aliyun.com/nexus/content/groups/public/

![在这里插入图片描述](https://img-blog.csdnimg.cn/40a5568801384e3091238f6ba9b0b339.png)

maven-public 中可以修改拉取依赖的优先级顺序

![在这里插入图片描述](https://img-blog.csdnimg.cn/10d782dde3b74a16966da7a2187e5428.png)