---
title: Maven基础
date: {{ date }}
categories:
- 工具
- Maven
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

## Mirror

mirror相当于拦截器，将 `mirrorOf` 中匹配的仓库ID请求到新的地址

注意：若jar包不存在阿里云，不能配置阿里云的 `mirrorOf` 为*，因为会重定向到阿里云仓库导致找不到jar包

```xml
<!-- 配置Mirror: central请求私服，其他的请求aliyun -->
<mirrors>
  <mirror>
    <id>thirdparty</id>
    <name>internal nexus repository</name>
    <url>http://xxxx/repository/sit-public/</url>
    <mirrorOf>central</mirrorOf>
  </mirror>
  <mirror>
    <id>nexus</id>
    <name>internal nexus repository</name>
    <url>https://maven.aliyun.com/repository/public</url>
    <mirrorOf>*,!central</mirrorOf>
  </mirror>
</mirrors>

<!-- 定义私服的仓库地址 -->
<profiles>
  <profile>
    <id>pro</id>
    <repositories>
      <repository>
        <id>central</id>
        <url>http://xxxx/repository/sit-public/</url>
        <releases>
          <enabled>true</enabled>
          <updatePolicy>always</updatePolicy>
          <checksumPolicy>warn</checksumPolicy>
        </releases>
        <snapshots>
          <enabled>true</enabled>
          <updatePolicy>always</updatePolicy>
          <checksumPolicy>warn</checksumPolicy>
        </snapshots>
      </repository>
    </repositories>
  </profile>
</profiles>
```

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

