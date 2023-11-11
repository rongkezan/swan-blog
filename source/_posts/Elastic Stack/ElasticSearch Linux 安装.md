---
title: ElasticSearch Linux 安装
date: {{ date }}
categories:
- Elastic Stack
---

## ElasticSearch 安装

> Linux版本：CentOS 7.9
>
> 前提条件：Java8+

### 1. 导入ElasticSearch公钥

```sh
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

### 2. 创建ElasticSeatch仓库

- 创建一个新的repo文件：`sudo vi /etc/yum.repos.d/elasticsearch.repo`。
- 然后添加以下内容：

```sh
[elasticsearch-7.x]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

### 3. 安装ElasticSearch

```sh
sudo yum install -y elasticsearch
```

### 4. 调整JVM堆栈大小

```sh
vim /etc/elasticsearch/jvm.options
```

将其调整为较低的值

```sh
-Xms4g
-Xmx4g
```

### 5. 调整ES配置

```sh
vim /etc/elasticsearch/elasticsearch.yml
```

允许公网访问

```sh
network.host: 0.0.0.0
http.port: 9200
```

单节点集群配置

```sh
cluster.initial_master_nodes: ["node-1"]
```

### 6. 启动并启动ElasticSearch

```sh
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```

### 7. 查看服务状态

```sh
sudo systemctl status elasticsearch
```

访问 http://ip:9200 出现以下示例返回

```json
{
    "name": "DEV-NODE4-JP",
    "cluster_name": "elasticsearch",
    "cluster_uuid": "yenhvp4LSOaEZcYZIVUeeg",
    "version": {
        "number": "7.17.14",
        "build_flavor": "default",
        "build_type": "rpm",
        "build_hash": "774e3bfa4d52e2834e4d9d8d669d77e4e5c1017f",
        "build_date": "2023-10-05T22:17:33.780167078Z",
        "build_snapshot": false,
        "lucene_version": "8.11.1",
        "minimum_wire_compatibility_version": "6.8.0",
        "minimum_index_compatibility_version": "6.0.0-beta1"
    },
    "tagline": "You Know, for Search"
}
```