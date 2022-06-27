---
title: Kong
date: {{ date }}
categories:
- 中间件
- Kong
---

## Kong的安装

> Docker方式

### 1. 创建网络

```sh
docker network create kong-net
```

### 2. 新建挂载卷

```sh
docker volume create kong-volume
```

### 3. 启动Kong数据库容器

```sh
docker run -d --name kong-database \
  --network=kong-net \
  -p 5432:5432 \
  -v kong-volume:/var/lib/postgresql/data \
  -e "POSTGRES_USER=kong" \
  -e "POSTGRES_DB=kong" \
  -e "POSTGRES_PASSWORD=kongpass" \
  postgres:9.6
```

### 4. 初始化Kong数据

```sh
docker run --rm --network=kong-net \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_PASSWORD=kongpass" \
  -e "KONG_PASSWORD=test" \
 kong/kong-gateway:2.8.1.1-alpine kong migrations bootstrap
```

### 5. 启动kong-gateway容器

```sh
docker run -d --name kong-gateway \
  --network=kong-net \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_USER=kong" \
  -e "KONG_PG_PASSWORD=kongpass" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
  -e "KONG_ADMIN_GUI_URL=http://localhost:8002" \
  -e KONG_LICENSE_DATA \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 8444:8444 \
  -p 8002:8002 \
  -p 8445:8445 \
  -p 8003:8003 \
  -p 8004:8004 \
  kong/kong-gateway:2.8.1.1-alpine
```

### 6. 测试是否安装成功

```
浏览器访问
http://localhost:8001
http://localhost:8002
```

## Kong管理后台Konga安装

### 1. 新建挂载卷

```sh
docker volume create konga-volume
```

### 2. 启动Konga数据库容器

```sh
docker run -d --name konga-database \
  --network=kong-net \
  -p 5433:5432 \
  -v konga-volume:/var/lib/postgresql/data \
  -e "POSTGRES_USER=konga" \
  -e "POSTGRES_DB=konga" \
  -e "POSTGRES_PASSWORD=konga" \
  postgres:9.6
```

### 3. 初始Konga数据

```sh
docker run --rm --network=kong-net \
pantsel/konga:latest -c prepare -a postgres -u postgres://konga:konga@konga-database:5432/konga
```

### 4. 启动Konga容器

```sh
docker run -d -p 1337:1337 \
	--network kong-net \
	--name konga \
	-e "DB_ADAPTER=postgres" \
	-e "DB_URI=postgres://konga:konga@konga-database:5432" \
	-e "NODE_ENV=production" \
	-e "DB_PASSWORD=konga" \
	pantsel/konga
```

### 5. 注册登陆后建立连接并激活

![在这里插入图片描述](https://img-blog.csdnimg.cn/3a92f6cf363d4a809c0376c98a4217bf.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/facdd2627c274f7d909135dfd6e29667.png)

## 动态负载均衡配置

### 组件说明

service：服务，可以直接指向一个API服务节点（host参数设置为ip+port），也可以指定一个upstream实现负载均衡。简单来说，服务用于映射被转发的后端API的节点集合。

route：路由，它负责匹配实际请求，映射到service中。

upstream：对应一组API节点，实现负载均衡。

target：对应一个API节点。

<img src="https://img-blog.csdnimg.cn/f4c6e213e4bb4beeb1ef2e0a0ead5a62.png" alt="在这里插入图片描述" style="zoom:50%;" />

### 1. 配置upstreams

```sh
curl -X POST http://127.0.0.1:8001/upstreams --data "name=my-upstream"
```

### 2. 配置target

```sh
curl -X POST http://127.0.0.1:8001/upstreams/my-upstream/targets --data "target=192.168.0.108:14251" --data "weight=100"
```

### 3. 配置service

```sh
# name: service的名称
# host: upstream的名称
curl -X POST http://127.0.0.1:8001/services --data "name=my-service" --data "host=my-upstream"
```

### 4. 配置route

```sh
curl -X POST http://127.0.0.1:8001/services/my-service/routes --data "name=my-route" --data "paths[]=/pms"
```

## 鉴权配置

### Basic Auth

#### 1. 在service中增加一个插件basic-auth

> service维度的鉴权

![在这里插入图片描述](https://img-blog.csdnimg.cn/f40ace592054492faa81bc9b22a7d4c7.png)

#### 2. 增加一个consumer

![在这里插入图片描述](https://img-blog.csdnimg.cn/4e8be530d4b240968d8984fa17576705.png)

#### 3. 在登陆时增加basic-auth

![在这里插入图片描述](https://img-blog.csdnimg.cn/0afdec090b744d008b55799ac5cf27f6.png)

### JWT

#### 1. 添加JWT插件

![在这里插入图片描述](https://img-blog.csdnimg.cn/49542558f67c4ec38ac991bc5b6fd7bb.png)

#### 2. 消费者配置JWT

![在这里插入图片描述](https://img-blog.csdnimg.cn/437efb2e227a43cd947521e55d7ea7bb.png)

#### 3. 生成JWT密文

https://jwt.io/

key作为iss，secret作为盐

![在这里插入图片描述](https://img-blog.csdnimg.cn/fe688259069f4a95adc9532b7dafbaa3.png)

#### 4. 构造请求头

请求时加上请求头Authorization，内容为 `Bearer 密文`

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJpc3MiOiJjb3JTUTRteGM3QWlIUUVGaERPOEtzb043NXlXZnA3NSJ9._p2YA2v1oieEGWWcfHHJSqBhl5nAppfZHkBAGNDFZQA
```

## 限流

### 添加插件Rate-limiting

可以配置秒、分钟、小时、天等多个维度，可以接受多少请求

![在这里插入图片描述](https://img-blog.csdnimg.cn/be52d6b72f1d496baf03b143393a2776.png)
