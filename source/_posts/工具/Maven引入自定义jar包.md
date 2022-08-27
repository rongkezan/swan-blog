---
title: Maven引入自定义jar包
date: {{ date }}
categories:
- 工具
---

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



