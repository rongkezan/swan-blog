---
title: ElasticSearch Linux 安装
date: {{ date }}
categories:
- Elastic Stack
---

## ElasticSearch 8.0.11 安装

> Linux版本：CentOS 7.9
>
> 无法使用root用户启动 elasticsearch

1. 打开下载页面下载 Linux x86_64 版本

https://www.elastic.co/cn/downloads/elasticsearch

2. 解压完 elasticsearch 后进入 bin 目录执行 `./elasticsearch` 得到如下示例输出

```
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  pBHGnG89W=H2L+efwkbj

ℹ️  HTTP CA certificate SHA-256 fingerprint:
  31841c39eb4d8b59ddd25f30f1cc049f0fcecc262659e0d3d5a5675ccbdb6187

ℹ️  Configure Kibana to use this cluster:
• Run Kibana and click the configuration link in the terminal when Kibana starts.
• Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjExLjAiLCJhZHIiOlsiMTAuMTEuMC44OjkyMDAiXSwiZmdyIjoiMzE4NDFjMzllYjRkOGI1OWRkZDI1ZjMwZjFjYzA0OWYwZmNlY2MyNjI2NTllMGQzZDVhNTY3NWNjYmRiNjE4NyIsImtleSI6IlJ5a0h3b3NCUm1qVWlWb1hCbFVpOlRybU92TEo3VGFPZEc0dEh5d282YlEifQ==

ℹ️  Configure other nodes to join this cluster:
• On this node:
  ⁃ Create an enrollment token with `bin/elasticsearch-create-enrollment-token -s node`.
  ⁃ Uncomment the transport.host setting at the end of config/elasticsearch.yml.
  ⁃ Restart Elasticsearch.
• On other nodes:
  ⁃ Start Elasticsearch with `bin/elasticsearch --enrollment-token <token>`, using the enrollment token that you generated.
```

首次启动 elasticsearch 时，会自动进行以下安全配置：

- 为传输层和HTTP层生成TLS证书和密钥
- TLS配置设置被写入 `elasticsearch.yml`
- 为 `elastic` 用户生成密码
- 为 `Kibana` 生成一个注册令牌

3. 如需让elasticsearch后台启动，执行以下命令

```sh
nohup ./elasticsearch &
```

## Kibana 安装

1. 打开下载页面下载 Linux x86_64 版本

https://www.elastic.co/cn/downloads/kibana

2. 解压完 kibana 后进入 config 目录修改 `kibana.yml` 使其可以外网访问

```yaml
server.port: 5601
server.host: "0.0.0.0"
```

3. 进入 bin 目录执行 `./kibana` 访问地址 http://ip:5601
4. 如需让Kibana后台启动，执行以下命令

```sh
nohup ./kibana &
```

