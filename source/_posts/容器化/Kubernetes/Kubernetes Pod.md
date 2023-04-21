---
title: Kubernetes Pod
date: {{ date }}
categories:
- Kubernetes
tags:
- Kubernetes
---

## Pod

pause：只要有Pod这个容器就会被启动

在同一个Pod中：共享网络栈、共享存储卷



ReplicationController用来确保容器应用的副本数始终保持在用户定义的副本数

ReplicaSet 与 ReplicationController 没有本质上的不同，只是ReplicaSet支持集合式的Selector

虽然ReplicaSet可以独立使用，但是建议使用 Deployment 来自动管理 ReplicaSet



HPA（Horizontal Pod Authscaling）：在V1版本中仅支持根据Pod的CPU利用率扩缩容，在vlalpha版本中，支持根据内存和用户自定义的metric扩缩容。

StatefulSet：解决有状态服务的问题

- 稳定的持久化存储：Pod重新调度后还是能访问到相同的持久化数据，基于PVC实现
- 稳定的网络标志：Pod重新调度后其PodName和HostName不变，基于Headless Service实现
- 有序部署，有序扩展：Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次进行，基于init container实现
- 有序收缩，有序删除：即从N-1到0



DaemonSet：确保全部或者一些Node上运行一个Pod的副本，当有Node加入集群时，也会为他们新增一个Pod，当有Node从集群移除时，这些Pod也会被回收，删除DaemonSet将会删除他创建的所有Pod。



Job：负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束。

Cron Job管理基于时间的Job，即：

- 在给定时间点只运行一次
- 周期性地在给定时间点运行