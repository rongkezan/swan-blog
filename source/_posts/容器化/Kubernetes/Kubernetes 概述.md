---
title: Kubernetes 概述
date: {{ date }}
categories:
- Kubernetes
tags:
- Kubernetes
---

Kubernetes 特点：轻量级、消耗资源小、弹性伸缩、负载均衡IPVS

## 架构图

<img src="https://img-blog.csdnimg.cn/4436a5ab27694513912c0c4ee8093b8a.png" alt="在这里插入图片描述" style="zoom:75%;" />

## 组件说明

- ApiServer：所有服务访问统一入口
- ReplicationController：维持副本期望数目
- Scheduler：负责接收任务，选择合适的节点分配任务
- ETCD
  - 可信赖的分布式键值存储服务，存储K8S集群的所有重要信息
  - v2版本会把所有的数据全部存储在内存中
  - v3版本会引入一个卷的持久化操作
  - 定时总量备份+持续增量备份
  - <img src="https://img-blog.csdnimg.cn/fdc12aabb32f42caa621c2d0706672ca.png" alt="在这里插入图片描述" style="zoom:50%;" />

- Kubelet：直接跟容器引擎交互实现容器的生命周期管理
- KubeProxy：负责写入规则至 IpTables、IPVS 实现服务映射访问的

## 重要插件

- CoreDNS：可以为集群中的SVC创建一个域名IP的对应关系解析
- Dashboard：给K8S集群提供一个B/S结构的访问体系
- IngressController：官方实现了4层代理，Ingress可以实现7层代理
- Federation：提供一个可以跨集群中心多K8S统一管理功能
- Prometheus：提供一个K8S集群的监控能力
- ELK：提供K8S集群日志统一分析接入平台

