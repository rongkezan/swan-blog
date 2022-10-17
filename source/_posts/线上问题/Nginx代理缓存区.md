---
title: Nginx代理缓存区
date: {{ date }}
categories:
- 线上问题
---

## 问题描述

浏览器出现

```
net::ERR_CONTENT_LENGTH_MISMATCH 206 (Partial Content)
```

原因是Nginx代理之后会有相应的代理缓存区，缓存区默认只有几十K，某些版本的nginx默认设置中没有相关处理，导致部分文件代理是会出现加载不全的现象，其实不仅仅是JS文件。只是因为框架的JS文件略大，所以经常出现类似问题。

## 解决方案

在Nginx配置中，http块下加上如下配置

```
proxy_buffer_size 128k;
proxy_buffers   32 128k;
proxy_busy_buffers_size 128k;
```

重启nginx即可