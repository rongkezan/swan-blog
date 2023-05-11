---
title: Elasticsearch 基本概念
date: {{ date }}
categories:
- Elasticsearch
---

## 节点 Node

一个节点就是一个 Elasticsearch 实例，可以理解为一个 es 进程

注意：一个节点 ≠ 一台服务器

查看节点接口

```
http://localhost:9200/_cat/nodes?v
```

## 角色 Role

- **主节点（active master）**：一般指活跃的主节点，一个集群中只能有一个，主要作用是对集群的管理。
- **候选节点（master-eligible）**：当主节点发生故障时，参与选举，也就是主节点的替代节点。
- **数据节点（data node）**：数据节点保存包含已编入索引的文档的分片。数据节点处理数据相关操作，如 CRUD、搜索和聚合。这些操作是 I/O 密集型、内存密集型和 CPU 密集型的。监控这些资源并在它们过载时添加更多数据节点非常重要。
- **预处理节点（ingest node）**：预处理节点有点类似于logstash的消息管道，所以也叫ingest pipeline，常用于一些数据写入之前的预处理操作。

准确的说，应该叫节点角色，是区分不同功能节点的一项服务配置，配置方法为

```yaml
# 如果 node.roles 为缺省配置，那么当前节点具备所有角色
node.roles: [ 角色1, 角色2, xxx ]
```

## 集群 Cluster

### 单体服务

所有的的服务依赖于同一个节点

**缺点**

- 处理能力（包括吞吐量、并发能力、和算力等）有限，当业务量不断增加时，单体服务无法满足。
- 所有的服务依赖于同一个节点，当该节点出现故障，服务就完全不可用，风险高，可用性差。

### 集群的概念

一张图理解什么是集群

| [![img](https://www.elastic.org.cn/upload/2022/11/5.jpg)](https://www.elastic.org.cn/upload/2022/11/5.jpg) | [![img](https://www.elastic.org.cn/upload/2022/11/6.jpg)](https://www.elastic.org.cn/upload/2022/11/6.jpg) |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| **单体应用**                                                 | **分布式应用**                                               |

### 自动发现

ES 是自动发现的，即零配置，开箱即用，无需任何网络配置，Elasticsearch 将绑定到可用的环回地址并扫描本地端口9300到9305连接同一服务器上运行的其他节点，自动形成集群。此行为无需进行任何配置即可提供自动集群服务。

### 核心配置

- **network.host**：即提供服务的ip地址，一般配置为本节点所在服务器的内网地址，此配置会导致节点由开发模式转为生产模式，从而触发引导检查。
- **network.publish_host**：即提供服务的ip地址，一般配置为本节点所在服务器的公网地址
- **http.port**：服务端口号，默认 9200，通常范围为 9200~9299
- **transport.port**：节点通信端口，默认 9300，通常范围为 9300~9399
- **discovery.seed_hosts**：此设置提供集群中其他候选节点的列表，并且可能处于活动状态且可联系以播种[发现过程](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-hosts-providers.html)。每个地址可以是 IP 地址，也可以是通过 DNS 解析为一个或多个 IP 地址的主机名。
- **cluster.initial_master_nodes**：指定集群初次选举中用到的候选节点，称为集群引导，只在第一次形成集群时需要，如过配置了 network.host，则此配置项必须配置。重新启动节点或将新节点添加到现有集群时不要使用此设置。

**基于内网配置集群**

[![img](https://www.elastic.org.cn/upload/2022/11/7.jpg)](https://www.elastic.org.cn/upload/2022/11/7.jpg)

**基于公网配置集群**

[![img](https://www.elastic.org.cn/upload/2022/11/8.jpg)](https://www.elastic.org.cn/upload/2022/11/8.jpg)

### 集群的健康值检查

#### 健康状态

- 绿色：所有分片都可用
- 黄色：至少有一个副本不可用，但是所有主分片都可用，此时集群能提供完整的读写服务，但是可用性较低。
- 红色：至少有一个主分片不可用，数据不完整。此时集群无法提供完整的读写服务。集群不可用。

[![9-1668331688515](https://www.elastic.org.cn/upload/2022/11/9-1668331688515.jpg)](https://www.elastic.org.cn/upload/2022/11/9-1668331688515.jpg)

[9-1668331688515](https://www.elastic.org.cn/upload/2022/11/9-1668331688515.jpg)



**新手误区：对不同健康状态下的可用性描述，集群不可用指的是集群状态为红色，无法提供完整读写服务，而不代表无法通过客户端远程连接和调用服务。**

#### 健康值检查

**方法一：_cat API**

```json
GET _cat/health
```

返回结果如下

[![img](https://www.elastic.org.cn/upload/2022/11/10.jpg)](https://www.elastic.org.cn/upload/2022/11/10.jpg)

**方法二：_cluster API**

```json
GET _cluster/health
```

返回结果如下

[![img](https://www.elastic.org.cn/upload/2022/11/11.jpg)](https://www.elastic.org.cn/upload/2022/11/11.jpg)

\## 6.6 集群的故障诊断

集群常见故障诊断手段通常为：通过检查集群的健康状态，是否有节点未加入或者脱离集群，以及是否有异常状态的分片。可采取以下 API 完成对集群的故障诊断。

#### Cat API

**常用APIs：**

- _cat/indices?health=yellow&v=true：查看当前集群中的所有索引
- _cat/health?v=true：查看健康状态
- _cat/nodeattrs：查看节点属性
- _cat/nodes?v：查看集群中的节点
- _cat/shards：查看集群中所有分片的分配情况

#### Cluster API

- _cluster/allocation/explain：可用于诊断分片未分配原因
- _cluster/health/ ：检查集群状态

#### 索引未分配的原因

- ALLOCATION_FAILED: 由于分片分配失败而未分配
- CLUSTER_RECOVERED: 由于完整群集恢复而未分配.
- DANGLING_INDEX_IMPORTED: 由于导入悬空索引而未分配.
- EXISTING_INDEX_RESTORED: 由于还原到闭合索引而未分配.
- INDEX_CREATED: 由于API创建索引而未分配.
- INDEX_REOPENED: 由于打开闭合索引而未分配.
- NEW_INDEX_RESTORED: 由于还原到新索引而未分配.
- NODE_LEFT: 由于承载它的节点离开集群而取消分配.
- REALLOCATED_REPLICA: 确定更好的副本位置并取消现有副本分配.
- REINITIALIZED: 当碎片从“开始”移回“初始化”时.
- REPLICA_ADDED: 由于显式添加了复制副本而未分配.
- REROUTE_CANCELLED: 由于显式取消重新路由命令而取消分配.

## 索引 Index

ES中的索引可以理解为 Mysql 中的表，但这只是类比去理解，索引并不等于表。

在 ES 中，索引在不同的特定条件下可以表示三种不同的意思：

- 表示源文件数据：当作数据的载体，即类比为数据表
- 表示索引文件：以加速查询检索为目的而设计和创建的数据文件，通常承载于某些特定的数据结构，如哈希，FTS等
- 表示创建数据的动作：通常说创建或添加一条数据，在 ES 表述为索引一条数据或索引一条文档

索引的组成部分：

- alias：索引别名
- settings：索引设置，常见的设置如分片和副本数量等
- mapping：映射，定义了索引中包含哪些字段，以及字段的类型、长度、分词器等。

索引的不可变性

- 索引名称
- 主分片数量
- 字段类型

## 文档 Document

Index 里面单条的记录称为 Document（文档）。许多条 Document 构成了一个 Index。

### 元数据 meta data

所有的元字段均以下划线开头，为系统字段

- _index：索引名称
- _id：文档ID
- _version：版本号
- _seq_no：索引级别的版本号，索引中所有文档共享一个 _seq_no
- _primary_term：是一个整数，每当 Primary Shard 发生重新分配时，比如节点重启，Primary 选举或重新分配等， _primary_term会递增1.

### 源数据  source data

业务数据，最终写入的用户数据

## 分片 Shard

### 分片的基本概念

如过用一句话来概括，分片可以理解为 索引的碎片。并且所有碎片都是可以无限复制的

### 分片的种类

- 主分片（primary shard）
- 副本分片（replica shard）

### 分片的基本策略

- 一个索引包含一个或多个分片，在7.0之前默认五个主分片，每个主分片一个副本；在7.0之后默认一个主分片。副本可以在索引创建之后修改数量，但是主分片的数量一旦确定不可修改，只能创建索引
- 每个分片都是一个Lucene实例，有完整的创建索引和处理请求的能力
- ES会自动再nodes上做分片均衡 shard reblance
- 一个doc不可能同时存在于多个主分片中，但是当每个主分片的副本数量不为一时，可以同时存在于多个副本中。
- `主分片和其副本分片`不能同时存在于同一个节点上。
- `完全相同的副本`不能同时存在于同一个节点上。

### 分片的作用和意义

- 高可用性：提高分布式服务的高可用性。
- 提高性能：提供系统服务的吞吐量和并发响应的能力
- 易扩展：当集群的性能不满足业务要求时，可以方便快速的扩容集群，而无需停止服务。