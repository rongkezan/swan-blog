---
title: Filebeats
date: {{ date }}
categories:
- Elastic Stack
---

Filebeat是一个轻量级的日志收集器，它可以将日志数据从各种来源（例如文件、系统日志、网络数据流等）收集并发送到Elasticsearch或Logstash进行进一步的处理和分析。

下载地址

https://www.elastic.co/cn/downloads/beats/filebeat

修改 `filebeat.yml` 配置

```yaml
filebeat.inputs:
- type: log
  id: my-log-id
  enabled: true
  paths:
    - /var/log/*.log
```

运行 filebeat

```sh
filebeat -e -c filebeat.yml
```

可以通过 Kibana -> Observability -> Logs -> Stream 查看日志

搜索日志格式

```
message:"<keyword>"
```







