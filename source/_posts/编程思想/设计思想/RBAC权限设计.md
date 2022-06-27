---
title: RBAC权限设计
date: {{ date }}
categories:
- 编程思想
- 设计思想
---

## RBAC 概述

RBAC模型（Role-Based Access Control：基于角色的访问控制）

权限与角色相关联，用户通过成为适当角色的成员而得到这些角色的权限。

权限：具备操作某个事务的能力

角色：一系列权限的集合

## RABC 数据库表设计

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327205902709.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)