---
title: Maven
date: {{ date }}
categories:
- 工具
- Maven
---

# Maven

## Maven 加载顺序

从左到右依次

![img](https://img-blog.csdnimg.cn/20200623083158201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hfd2Zlbmc=,size_16,color_FFFFFF,t_70)

1. 项目中的pom.xml文件中的两种引用

```xml
<repositories>
  <repository>
    <id>central</id>
    <url>http://${nexus.url}/repository/maven-public/</url>
  </repository>
</repositories>

<profiles>
  <profile>
    <!-- 通过使用profiles选择local 生效 -->
    <id>local</id> 
    <repositories>
      <repository>
        <id>central</id>
        <url>http://${nexus.url}/repository/maven-public/</url>
        <!-- 开发环境设置releases版本也更新本地jar -->
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

2. maven 中的setting.xml文件中的两种引用

```xml
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

## Snapshot Release

快照版本和正式版本的主要区别在于，本地获取这些依赖的机制有所不同。

构建正式版本（release）时，第一次构建的时候会把该包从远程仓库中下载到本地仓库缓存，以后再次构建都不会去访问远程仓库了，你只有在重新发布的时候升级版本，才能使用到你最新添加的功能。

构建快照版本（snapshots）时，会优先去下载远程仓库最新的包，Maven的Repository的时候中有个配置项，可以配置对于SNAPSHOT版本向远程仓库中查找的频率。频率共有四种，分别是always、daily、interval、never。

- always：每次都去远程仓库查看是否有更新

- daily：只在第一次的时候查看是否有更新，当天的其它时候则不会查看

- interval：允许设置一个分钟为单位的间隔时间，在这个间隔时间内只会去远程仓库中查找一次

- never：不会去远程仓库中查找（和正式版本的行为一样）

```xml
<profiles>
  <profile>
    <!-- 通过使用profiles选择local 生效 -->
    <id>local</id> 
    <repositories>
      <repository>
        <id>central</id>
        <url>http://${nexus.url}/repository/maven-public/</url>
        <!-- 开发环境设置releases版本更新本地jar -->
        <releases> 
          <enabled>true</enabled>
          <updatePolicy>always</updatePolicy>
          <checksumPolicy>warn</checksumPolicy>
        </releases>
        <!-- 开发环境设置snapshots版本更新本地jar -->
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

## Maven 打包源码

在 pom.xml 中添加以下内容，可以在maven生成jar的同时生成sources包

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-source-plugin</artifactId>
	<version>3.0.0</version>
	<!-- 绑定source插件到Maven的生命周期,并在生命周期后执行绑定的source的goal -->
	<executions>
		<execution>
			<!-- 绑定source插件到Maven的生命周期 -->
			<phase>compile</phase>
			<!--在生命周期后执行绑定的source插件的goals -->
			<goals>
				<goal>jar-no-fork</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

另一种写法

```xml
<!-- Source attach plugin -->
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-source-plugin</artifactId>
	<executions>
		<execution>
			<id>attach-sources</id>
			<goals>
				<goal>jar</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

## Maven 跳过测试

```shell
mvn install -Dmaven.test.skip=true
```

## Maven配置

### Repository 和 Mirror

Mirror可以作为Repositoty的加速镜像

可以通过 `mirrorOf` 指定需要加速的仓库

例：

`settings.xml`

```xml
<mirror>
  <id>aliyunmaven</id>
  <mirrorOf>*</mirrorOf>
  <name>阿里云公共仓库</name>
  <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

`pom.xml`

```xml
<repositories>
	<repository>
		<id>rep1</id>
		<url>https:/test1.org/maven/</url>
	</repository>
	<repository>
		<id>rep2</id>
		<url>https://test2.com/releases/</url>
	</repository>
</repositories>
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

<!-- 配置远程私有仓库 -->
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

