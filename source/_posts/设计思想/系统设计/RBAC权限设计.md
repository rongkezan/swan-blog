---
title: RBAC权限设计
date: {{ date }}
categories:
- 系统设计
---

## RBAC 概述

RBAC模型（Role-Based Access Control：基于角色的访问控制）

权限与角色相关联，用户通过成为适当角色的成员而得到这些角色的权限。

权限：具备操作某个事务的能力

角色：一系列权限的集合

## RBAC 数据库表设计

用户表：user_id, user_name

角色表：role_id, role_name

权限表：permission_id, permission_key

用户角色关联表：user_id, role_name

角色权限关联表：role_name, permission_key

