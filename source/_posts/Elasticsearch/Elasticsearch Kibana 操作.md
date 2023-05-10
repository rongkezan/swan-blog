---
title: Elasticsearch Kibana 操作
date: {{ date }}
categories:
- Elasticsearch
---

## 索引操作
建立索引
```json
PUT /索引库名
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  }
}
```
查询索引
```json
GET demo1
```
删除索引
```json
DELETE demo1
```
查询索引是否存在
```json
HEAD demo1
```
## 映射配置
映射是定义文档的过程，文档包含哪些字段，这些字段是否保存，是否索引，是否分词等


创建映射
```json
PUT 索引库名/_mapping/类型名
{
  "properties": {
    "字段名":{
      "type": "类型",
      "index": true,
      "store": false,
      "analyzer": "分词器"
    }
  }
}
```
查看映射关系
```json
GET /索引库名/_mapping
```
## 增删改
新增数据，随机生成ID
```json
POST /索引库名/类型名
{
	"key": "value"
}
```
新增数据，指定ID
```json
POST /索引库名/类型名/id
{
	"key": "value"
}
```
根据id修改（PUT也可以用于新增）
```json
PUT /索引库名/类型名/3
{
	"key": "value"
}
```
根据id删除
```json
DELETE 索引库名/类型名/id
```
## 查询操作

基本语法

```java
GET /<index_name>/_search 
GET /_search 
```

- size：单次查询多少条文档，默认为 10
- from：起始文档偏移量。需要为非负数，默认为0
- timeout：指定等待每个分片响应的时间段。如果在超时到期之前未收到响应，则请求失败并返回错误。默认为无超时。

查询所有数据
```json
GET _search
{
  "query": {
    "match_all": {}
  }
}
```
查询单个索引库所有数据
```json
GET /索引库/_search
{
    "query":{
        "match_all": {}
    }
}
```
根据条件查询（分词）
```json
GET /索引库/_search
{
    "query":{
        "match": {
			"字段名": "关键字"
		}
    }
}
```
and查询（不分词）
```json
GET 索引库名/_search
{
  "query": {
    "match": {
      "字段名": {
        "query": "关键字",
        "operator": "and"
      }
    }
  }
}
```
多字段查询
```json
GET 索引库名/_search
{
  "query": {
    "multi_match": {
      "query": "关键字",
      "fields": ["字段1", "字段2"]
    }
  }
}
```
词条匹配，查询不分词的字段（除text类型以外的所有类型都是不分词的）
```json
GET 索引库名/_search
{
  "query": {
    "term": {
      "字段名": {
        "value": "关键字"
      }
    }
  }
}
```
结果过滤（包含），搜索结果只包含这些字段

```json
GET 索引库名/_search
{
  "_source": {
    "includes": ["字段1", "字段2"]
  },
  "query": {
    "match": {
      "字段名": "关键字"
    }
  }
}
```
结果过滤（排除），搜索结果排除这些字段
```json
GET 索引库名/_search
{
  "_source": {
    "excludes": ["字段1", "字段2"]
  },
  "query": {
    "match": {
      "字段名": "关键字"
    }
  }
}
```
模糊查询，有一定的容错性
```json
GET 索引库名/_search
{
  "query": {
    "fuzzy": {
      "字段名": "关键字"
    }
  }
}
```
范围查询
```json
GET 索引库名/_search
{
  "query": {
    "range": {
      "字段名": {
        "gte": 1000,
        "lte": 3000
      }
    }
  }
}
```
布尔查询（must），查询条件需同时成立
```json
GET 索引库名/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
          	"字段1": "关键字"
          }
        },
        {
          "range": {
            "字段2": {
              "gte": 1000
            }
          }
        }
      ]
    }
  }
}
```
布尔查询（should），查询条件只需满足一个
```json
GET 索引库名/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
          	"字段1": "关键字"
          }
        },
        {
          "range": {
            "字段2": {
              "gte": 1000
            }
          }
        }
      ]
    }
  }
}
```
布尔查询（must_not），满足条件不查询
```json
GET 索引库名/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "match": {
          	"字段1": "关键字"
          }
        },
        {
          "range": {
            "字段2": {
              "gte": 1000
            }
          }
        }
      ]
    }
  }
}
```
布尔查询配合filter，不影响得分
```json
GET 索引库名/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "字段1": "关键字"
          }
        }
      ],
      "filter": {
          "range": {
            "字段2": {
              "gte": 1000
            }
        }
      }
    }
  }
}
```
查询排序
```json
GET 索引库名/_search
{
  "query": {
    "match": {
      "字段1": "关键字"
    }
  },
  "sort": [
    {
      "字段2": {
        "order": "desc"
      }
    }
  ]
}
```
分页查询
```json
GET 索引库名/_search
{
  "query": {
    "match": {
      "字段1": "关键字"
    }
  },
  "from": 0,
  "size": 20
}
```
## 聚合
聚合可以让我们极其方便的实现对数据的统计、分析。

根据词条聚合
```json
GET 索引库名/_search
{
  "size": 0, 
  "aggs": {
    "聚合名称": {
      "terms": {
        "field": "字段名"
      }
    }
  } 
}
```
根据词条聚合，并计算平均值

```json
GET 索引库名/_search
{
  "size": 0, 
  "aggs": {
    "聚合名称": {
      "terms": {
        "field": "字段1"
      },
      "aggs": {
        "子聚合名称": {
          "avg": {
            "field": "字段2"
          }
        }
      }
    }
  } 
}
```