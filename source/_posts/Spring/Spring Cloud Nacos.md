---
title: Spring Cloud Nacos
date: {{ date }}
categories:
- Spring
tags:
- Spring Cloud
---

## Nacos

### Nacos 概念

命名空间（Namespace）：代表不同环境，如开发、测试、生产环境

配置分组（Group）：代表某项目

配置集（Data ID）：每个项目下有若干个工程，每个配置集就是一个工程的主配置文件

### Nacos 单机部署

从官网下载 Last Release 版本

https://github.com/alibaba/nacos

在bin文件夹下以standalone模式启动

```sh
# windows
startup.cmd -m standalone
# linux
./startup.sh -m standalone
```

进入控制台，账密：nacos

http://localhost:8848/nacos/index.html

### Nacos 集群部署

#### Nacos 数据库配置

在自己的数据库中新建数据库，名为 `nacos_config`

进入目录 `nacos/conf` 

找到文件 `nacos.mysql.sql` ，复制其内容建表

找到文件 `application.properties` ， 修改其内容

```properties
### Count of DB:
db.num=1
### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
db.user.0=nacos
db.password.0=nacos
```

#### Nacos 集群配置

复制 `nacos/conf` 下的 `cluster.conf.example` 为 `cluster.conf`

配置集群信息

```
172.23.184.234:8848
172.23.184.238:8848
```

### Nacos 权限配置

> 基于nacos2.3.2版本

```properties
# 开启鉴权
nacos.core.auth.enabled=true
# nacos登录的username,password
nacos.core.auth.server.identity.key=nacos
nacos.core.auth.server.identity.value=nacos
# 默认token，需要32位以上
nacos.core.auth.plugin.nacos.token.secret.key=bmFjb3NfMjAyNDAxMTBfc2hpZ3poX25hY29zX3Rva2Vu
```

#### Nginx 配置

> upstream 名称不能用下划线，nginx无法识别 

```json
http {    
    upstream nacosCluster {
        server 172.23.184.238:8848;
        server 172.23.184.234:8848;
    }
	server {
        listen      80;
        server_name nacos.demo.com;

        location /nacos {
            proxy_pass  http://nacosCluster;
            if ($request_method = 'OPTIONS') {
                return 204;
            }
        }
    }
}
```

### Nacos 解决配置文件丢失问题

当 nacos 配置数据库，集群启动后发现原来在单机上的配置文件丢失了

解决方式：停止nacos服务，将原来那台nacos单机启动即可获取数据

### Nacos 配置中心

导入依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  <version>2.2.5.RELEASE</version>
</dependency>
```

配置 bootstrap.yml

1. 仅有一个主配置文件，在Nacos服务端配置文件名为 {spring.application.name}.{file-extension}

```yaml
spring:
  application:
    name: nacos-demo
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: 2cd03b3d-4c75-4ceb-ac89-2673fc2032d9
        file-extension: yml
        group: DEV_GROUP
```

2. 有多个配置文件，除了上述的主配置文件外还可以指定额外配置文件

```yaml
spring:
  application:
    name: nacos-demo
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: 2cd03b3d-4c75-4ceb-ac89-2673fc2032d9
        file-extension: yml
        group: DEV_GROUP
        extension-configs:
          - data-id: nacos-example1.yml
            group: DEV_GROUP
            refresh: true
          - data-id: nacos-example2.yml
            group: DEV_GROUP
            refresh: true
```

Nacos 服务端配置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210404190900874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

1. nacos-demo.yml

```yaml
server:
	port: 8001
```

2. nacos-example1.yml

```yaml
common:
	name: zhangsan
```

3. nacos-example1.yml

```yaml
common:
	age: 18
```

测试类

```java
@RestController
@RequestMapping("config")
@RefreshScope
public class ConfigController {

    @Value("${common.name}")
    private String name;

    @Value("${common.age}")
    private Integer age;

    @GetMapping("get")
    public String get(){
        return name + "," + age;
    }
}
```

### Nacos 服务发现 (OpenFeign)

导入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
	<version>2.2.5.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
	<version>2.2.9.RELEASE</version>
</dependency>
```

配置

```yaml
spring:
  profiles: prod
  cloud:
    nacos:
      username: nacos
      password: nacos
      discovery:
        server-addr: http://127.0.0.1:8848
        namespace: 2dd4a07a-9860-4ba0-ae1b-7c9dda2cdc25
```

在启动类上增加注解 `@EnableFeignClients`

创建客户端调用类

```java
@FeignClient("nft-card")
public interface MyFeignClient {
    @GetMapping("test")
    String check();
}
```

### Nacos 服务发现 (Dubbo)

导入依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  <version>2.2.5.RELEASE</version>
</dependency>
<dependency>
  <groupId>org.apache.dubbo</groupId>
  <artifactId>dubbo-registry-nacos</artifactId>
  <version>2.7.8</version>
</dependency>
<dependency>
  <groupId>org.apache.dubbo</groupId>
  <artifactId>dubbo-spring-boot-starter</artifactId>
  <version>2.7.8</version>
</dependency>
```

生产者配置

```yaml
server:
  port: 8001
spring:
  application:
    name: sardine-nacos-provider
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yml
dubbo:
  protocol:
    name: dubbo
    port: 20881
  registry:
    address: nacos://127.0.0.1:8848
  application:
    name: ${spring.application.name}
```

消费者配置

```yaml
server:
  port: 8002
spring:
  application:
    name: sardine-nacos-consumer
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yml
dubbo:
  protocol:
    name: dubbo
    port: 20882
  registry:
    address: nacos://127.0.0.1:8848
  application:
    name: ${spring.application.name}
```

生产者启动类上加注解

```java
@EnableDubbo
```

消费者加入生产者依赖

```xml
<dependency>
  <groupId>com.demo</groupId>
  <artifactId>demo-api</artifactId>
  <version>1.0.0</version>
</dependency>
```

消费者调用生产者

```java
@DubboReference
private ProviderService providerService;
```

