---
title: Maven
date: {{ date }}
categories:
- 工具
---

## Maven 加载顺序

从左到右依次

![img](https://img-blog.csdnimg.cn/20200623083158201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hfd2Zlbmc=,size_16,color_FFFFFF,t_70)

## Snapshot Release

快照版本和正式版本的主要区别在于，本地获取这些依赖的机制有所不同。

构建正式版本（release）时，第一次构建的时候会把该包从远程仓库中下载到本地仓库缓存，以后再次构建都不会去访问远程仓库了，你只有在重新发布的时候升级版本，才能使用到你最新添加的功能。

构建快照版本（snapshots）时，会优先去下载远程仓库最新的包，Maven的Repository的时候中有个配置项，可以配置对于SNAPSHOT版本向远程仓库中查找的频率。频率共有四种，分别是always、daily、interval、never。

- always：每次都去远程仓库查看是否有更新

- daily：只在第一次的时候查看是否有更新，当天的其它时候则不会查看

- interval：允许设置一个分钟为单位的间隔时间，在这个间隔时间内只会去远程仓库中查找一次

- never：不会去远程仓库中查找（和正式版本的行为一样）

## Maven Profile

可以使用 `-P` 指定profile

```sh
mvn groupId:artifactId:goal -P profile-1
```

Profiles可以在 Maven settings 中通过 `<activeProfiles>` 激活

```xml
<settings>
  ...
  <activeProfiles>
    <activeProfile>profile-1</activeProfile>
  </activeProfiles>
  ...
</settings>
```

## Maven 打包源码插件

在 pom.xml 中添加以下内容，可以在maven生成jar的同时生成sources包

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-source-plugin</artifactId>
      <version>3.0.0</version>
      <executions>
        <execution>
          <phase>compile</phase>
          <goals>
            <goal>jar-no-fork</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

## Maven 命令

```shell
# mvn install 跳过测试环节
mvn install -Dmaven.test.skip=true
```

## 项目中的Common模块打包

Common模块都**不加入**

```xml
<plugins>
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
</plugins>
```

打包的时候去父工程打包 `mvn clean install`

## 本地Jar包安装到Maven仓库

```sh
mvn install:install-file -DgroupId="com.abc" -DartifactId="mavenTest" -Dversion="1.0.0" -Dpackaging="jar" -Dfile="test.jar"
```

## Pom样例

```xml
<!-- 打包时上传源码 -->
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.3</version>
      <configuration>
        <source>1.8</source>
        <target>1.8</target>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-source-plugin</artifactId>
      <version>2.4</version>
      <executions>
        <execution>
          <id>attach-sources</id>
          <goals>
            <goal>jar-no-fork</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>

<!-- 配置远程上传依赖的私有仓库 -->
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <name>Nexus Release Repository</name>
    <url>{Your repository}</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <name>Nexus Snapshot Repository</name>
    <url>{Your repository}</url>
  </snapshotRepository>
</distributionManagement>
```

## Maven 引入自定义 Jar 包

1.将jar放到项目根目录（lib文件夹下）

![img](https://img2022.cnblogs.com/blog/1761926/202207/1761926-20220727145959787-69400310.png)

2.修改pom文件

```xml
<dependency>
    <groupId>*******</groupId>
    <artifactId>*******</artifactId>
    <version>1.1-SNAPSHOT</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/*******-1.1-20220629.060335-9.jar</systemPath>
</dependency>
```

3.修改package配置

```xml
<build>
    <resources>
        <!--将项目根目录下的lib文件中的jar包全部打入BOOT-INF/lib文件下-->
        <resource>
            <directory>lib</directory>
            <targetPath>BOOT-INF/lib/</targetPath>
            <includes>
                <include>**/*.jar</include>
            </includes>
            <filtering>false</filtering>
        </resource>
        <!--将项目根目录下的src/main/resources文件中的配置文件全部打入默认文件下-->
        <resource>
            <directory>src/main/resources</directory>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
                <include>**/*.yml</include>
            </includes>
            <filtering>false</filtering>
        </resource>
    </resources>
</build>
```

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

