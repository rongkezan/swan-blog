---
title: Elasticsearch Kibana 操作
date: {{ date }}
categories:
- Elastic Stack
---

## 索引操作

基本语法

```json
PUT <index_name>
```

创建索引时指定 `settings` ：例如指定分片数为3，副本数为2
```json
PUT <index_name>
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  }
}
```
查询索引
```json
GET <index_name>
```
删除索引
```json
DELETE <index_name>
```
查询索引是否存在
```json
HEAD <index_name>
```
## 文档操作

### 数据写入

#### Create

创建数据，数据存在则报错

基本语法

```json
PUT /<index_name>/_doc/<_id>?op_type=create
{
    "key": "value"
}
```

简化写法

```json
PUT /<index_name>/_create/<_id>
{
    "key": "value"
}
```

#### Index

创建或全量更新数据，更新整个 `_source` 中的 json 对象

基本语法

```json
PUT /<index_name>/_doc/<_id>?op_type=index
{
    "key": "value"
}
```

简化写法

```json
PUT /<index_name>/_doc/<_id>
{
    "key": "value"
}
```

#### 自动生成ID

不指定ID即会自动生成ID

基本语法

```json
POST /<index_name>/_doc
{
    "key": "value"
}
```

#### Update

修改局部字段或者数据

```json
POST /<index>/_update/<_id>
{
  "doc": {
    "key": "value"
  }
}
```

#### Delete

删除数据

```json
DELETE <index_name>/_doc/<_id>
```

通过条件删除数据

```json
POST /<index_name>/_delete_by_query
{
  "query": {
    "term": {
		"price": 6999
    }
  }
}
```

#### 批量写入数据

Create

```json
POST /_bulk 
{"create":{"_index":"goods","_id":"21"}}
{"name":"傻妞手机","content":"华人牌2060款手机傻妞","price":12999,"type":"手机","level":"超级手机","createtime":"2060-10-01T08:00:00Z","tags":["时光穿梭","真人模式","手机模式"]}
```

Index

```json
POST /_bulk
{"index":{"_index":"goods","_id":"2"}}
{"name":"太子手机","content":"太子手机是傻妞手机的克隆机","price":8999,"type":"手机","level":"超级手机","createtime":"2060-10-01T08:00:00Z","tags":["时光穿梭","手机模式"]}
```

Update

```json
POST goods/_bulk
{ "update": { "_index": "goods",  "_id": "2"} }
{ "doc" : {"price" : 6999} }
```

Delete

```json
POST /_bulk
{ "delete": { "_index": "goods",  "_id": "1" }}
```

### 数据查询

#### 基本查询

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
GET /<index_name>/_search
{
    "query":{
        "match_all": {}
    }
}
```
根据条件查询（分词）
```json
GET /<index_name>/_search
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
GET /<index_name>/_search
{
  "query": {
    "match": {
      "<field_name>": {
        "query": "<keyword>",
        "operator": "and"
      }
    }
  }
}
```
多字段查询
```json
GET /<index_name>/_search
{
  "query": {
    "multi_match": {
      "query": "<keyword>",
      "fields": ["field_1", "field_2"]
    }
  }
}
```
词条匹配，查询不分词的字段（除text类型以外的所有类型都是不分词的）
```json
GET /<index_name>/_search
{
  "query": {
    "term": {
      "<field_name>": {
        "value": "<keyword>"
      }
    }
  }
}
```
结果过滤（包含），搜索结果只包含这些字段

```json
GET /<index_name>/_search
{
  "_source": {
    "includes": ["field_1", "field_2"]
  },
  "query": {
    "match": {
      "<field_name>": "<keyword>"
    }
  }
}
```
结果过滤（排除），搜索结果排除这些字段
```json
GET /<index_name>/_search
{
  "_source": {
    "excludes": ["field_1", "field_2"]
  },
  "query": {
    "match": {
      "<field_name>": "<keyword>"
    }
  }
}
```
模糊查询，有一定的容错性
```json
GET /<index_name>/_search
{
  "query": {
    "fuzzy": {
      "<field_name>": "<keyword>"
    }
  }
}
```
范围查询
```json
GET /<index_name>/_search
{
  "query": {
    "range": {
      "<field_name>": {
        "gte": 1000,
        "lte": 3000
      }
    }
  }
}
```
布尔查询（must），查询条件需同时成立
```json
GET /<index_name>/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
          	"<field_1>": "<keyword>"
          }
        },
        {
          "range": {
            "<field_2>": {
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
GET /<index_name>/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
          	"<field_1>": "<keyword>"
          }
        },
        {
          "range": {
            "<field_2>": {
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
GET /<index_name>/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "match": {
          	"<field_1>": "<keyword>"
          }
        },
        {
          "range": {
            "<field_2>": {
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
GET /<index_name>/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "<field_1>": "<keyword>"
          }
        }
      ],
      "filter": {
          "range": {
            "<field_2>": {
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
GET /<index_name>/_search
{
  "query": {
    "match": {
      "<field_1>": "<keyword>"
    }
  },
  "sort": [
    {
      "<field_2>": {
        "order": "desc"
      }
    }
  ]
}
```
分页查询
```json
GET /<index_name>/_search
{
  "query": {
    "match": {
      "<field>": "<keyword>"
    }
  },
  "from": 0,
  "size": 20
}
```
#### 批量查询

```json
GET /_mget
{
  "docs": [
    {
      "_index": "<index_name>",
      "_id": "<_id>"
    },
    {
      "_index": "<index_name>",
      "_id": "<_id>"
    }
  ]
}
```

精简写法

```json
GET <index_name>/_mget
{
  "docs": [
    {
      "_id": "<_id>"
    },
    {
      "_id": "<_id>"
    }
  ]
}
```

#### 聚合查询

聚合可以让我们极其方便的实现对数据的统计、分析。

根据词条聚合
```json
GET /<index_name>/_search
{
  "size": 0, 
  "aggs": {
    "<agg_name>": {
      "terms": {
        "field": "<field_name>"
      }
    }
  } 
}
```
根据词条聚合，并计算平均值

```json
GET /<index_name>/_search
{
  "size": 0, 
  "aggs": {
    "<agg_name>": {
      "terms": {
        "field": "<field_name_1>"
      },
      "aggs": {
        "<sub_agg_name>": {
          "avg": {
            "field": "<field_name_2>"
          }
        }
      }
    }
  } 
}
```

## 映射配置

映射是定义文档的过程，文档包含哪些字段，这些字段是否保存，是否索引，是否分词等

### 查看映射

查看完整的索引 mapping

```json
GET /<index_name>/_mapping
```

查看索引中指定字段的 mapping

```json
GET /<index_name>/_mapping/field/<field_name>
```

### 自动映射 dynamic mapping

自动映射也叫动态映射，是 ES 在索引文档写入发生时自动创建 mapping 的一种机制。

ES 在创建索引之前，并不强制要求创建索引的 mapping，ES 会根据字段的值来推断字段类型，进而自动创建并指定索引类型。

下面是自动映射器推断字段类型的规则：自动映射器会尽可能的把字段映射为宽字段类型。

[![2.1](https://www.elastic.org.cn/upload/2023/04/2.1.jpg)](https://www.elastic.org.cn/upload/2023/04/2.1.jpg)

### mapping 使用禁忌

- ES 没有隐式类型转换
- ES 不支持类型修改
- 生产环境尽可能的避免使用 dynamic mapping

### 手动映射 Explicit mapping

手动映射也叫做显式映射，即：在索引文档写入之前，认为的创建索引并且指定索引中每个字段类型、分词器等参数。

```json
PUT /<index_name>
{
  "mappings": {
    "properties": {
      "field_a": {
        "<parameter_name>": "<parameter_value>"
      },
      ...
    }
  }
}
```

### 修改 mapping 属性

注意和创建 mapping 时的语法区别。

```java
PUT <index_name>/_mapping
{
  "properties": {
    "<field_name>": {
      "type": "text",	// 必须和原字段类型相同，切不许显式声明
      "analyzer":"ik_max_word",	// 必须和元原词器类型相同，切必须显式声明
      "fielddata": false
    }
  }
}
```

**注意**：并非所有字段参数都可以修改。

- 字段类型不可修改
- 字段分词器不可修改

## ES 数据类型 ★：field data type

### 概述

每个字段都有字段数据类型或字段类型。其大致分为两种：**会被分词的字段类型**和**不会被分词的字段类型。**

- 会被分词的类型：text、match_only_text 等。
- 不会被分词类型：keyword、数值类型等。

当然数据类型的划分可以分为很多种，比如按照`基本数据类型和复杂数据类型`来划分。

### ES 支持的数据类型

**注意**

- **标注颜色为浅灰色代表非常用数据类型**
- **标注** ★ **为非常重要的数据类型，会单独用一节课来讲**

### 基本数据类型 ★

- Numbers

  ：数字类型，包含很多具体的基本数据类型

  ![4.2.1](https://www.elastic.org.cn/upload/2023/04/4.2.1.jpg)

- **binary**：编码为 Base64 字符串的二进制值。

- **boolean**：即布尔类型，接受 true 和 false。

- **alias**：字段别名。

- **Keywords**：包含 **keyword** ★、constant_keyword 和 wildcard。

- **Dates**：日期类型，包括 **date** ★ 和 data_nanos，两种类型

### 对象关系类型（复杂类型）

- **object**：非基本数据类型之外，默认的 json 对象为 object 类型。
- **flattened**：单映射对象类型，其值为 json 对象。
- **nested** ★：嵌套类型。
- **join**：父子级关系类型。

### 结构化类型

- **Range**：范围类型，比如 long_range，double_range，data_range 等
- **ip**：ipv4 或 ipv6 地址
- **version**：版本号
- **murmur3**：计算和存储值的散列

### 聚合数据类型

- **aggregate_metric_double**：
- **histogram**：

### 文本搜索字段

- **text** ★：文本数据类型，用于全文检索。
- **annotated-text：**
- **completion** ★**：**
- **search_as_you_type：**
- **token_count：**

### 文档排名类型

- dense_vector：记录浮点值的密集向量。
- rank_feature：记录数字特征以提高查询时的命中率。
- rank_features：记录数字特征以提高查询时的命中率。

### 空间数据类型 ★

- **geo_point**：纬度和经度点。
- **geo_shape**：复杂的形状，例如多边形。
- **point**：任意笛卡尔点。
- **shape**：任意笛卡尔几何。

### 其他类型

- **percolator**：用Query DSL 编写的索引查询。

### 常见类型解释

#### Text 类型 ★★

当一个字段是要被全文搜索的，比如邮件内容、产品描述等长文本，这些字段应该使用 text 类型。设置 text 类型以后，字段内容会被分词，在生成倒排索引以前，字符串会被分析器分成一个一个词项。text 类型的字段不用于排序，很少用于聚合。（解释一下为啥不会为 text 创建正排索引：大量堆空间，尤其是在加载高基数 text 字段时。字段数据一旦加载到堆中，就在该段的生命周期内保持在那里。同样，加载字段数据是一个昂贵的过程，可能导致用户遇到延迟问题。这就是默认情况下禁用字段数据的原因）

总的来说，text 用于长文本字段，可以说是 ES 体系中最重要也是最常见的数据类型。

注意：

在基础课程中，仅需掌握 text 类型用于哪些应用场景下，以及用法即可，其深层次原理及全文检索相关内容，会在进阶课程中倒排索引的章节中体现。

总结：

- 应用的业务场景：全文检索。
- Text 类型基本声明。
- Text 类型会被分词。
- ES 默认情况下会为 Text 类型创建倒排索引。

**注意**：

在基础课程中，仅需掌握 text 类型用于哪些应用场景下，以及用法即可，其深层次原理及全文检索相关内容，会在进阶课程中倒排索引的章节中体现。

#### keyword 类型 ★

keyword使用序号映射存储它们的文档值以获得更紧凑的表示。 此映射的工作原理是根据其词典顺序为每个术语分配一个增量整数或*序数。*该字段的文档值仅存储每个文档的序数而不是原始术语，并使用单独的查找结构在序数和术语之间进行转换。

一般用于精确匹配和聚合字段，例如 ID、电子邮件地址、主机名、状态代码、邮政编码或标签。包括范围查找。和 term 查询一起使用频率是最高的。

- keyword 类型字段不会被分词。
- keyword 一般用于精确查找或者聚合字段
- keyword 类型超过阈值长度会直接被丢弃

概括来说，就是用于“不分词”的字段，不需要“模糊检索”的字段，所以常和 `term` 一起使用，用于精准查询。

#### Date 类型 ★

时间和日期类型是我们作为开发每天都会遇到的一种常见数据类型。和 Java 中有所不同，Elasticsearch 在索引创建之前并不是必须要创建索引的 mapping。

Elasticsearch 会根据你写入的字段的内容动态去判定字段的数据类型，不过这种自动映射的机制存在一些缺陷，比如在 Elasticsearch 中没有隐式类型转换，所以在自动映射的时候就会把字段映射为较宽的数据类型。比如你写入一个数字 50，系统就会自动给你映射成 long 类型，而不是 int 。一般企业中用于生产的环境都是使用手工映射，能保证按需创建以节省资源和达到更高的性能。

在 Elasticsearch 中，时间类型是一个非常容易踩坑的数据类型，通过一个例子向大家展示这个时间类型的用法和避坑指南

##### 案例

假如我们有如下索引 tax ，保存了一些公司的纳税或资产信息，单位为“万元”。当然这里面的数据是随意填写的。多少为数据统计的时间，当前这个例子里。索引达的含义并不重要。关键点在于字段的内容格式。我们看到date字段其中包含了多种日期的格式：“yyyy-MM-dd”，“yyyy-MM-dd”还有时间戳。如果按照 dynamic mapping，采取自动映射器来映射索引。我们自然而然的都会感觉字段应该是一个date类型。

```json
POST tax/_bulk
{"index":{}}
{"date": "2021-01-25 10:01:12", "company": "中国烟草", "ratal": 5700000}
{"index":{}}
{"date": "2021-01-25 10:01:13", "company": "华为", "ratal": 4034113.182}
{"index":{}}
{"date": "2021-01-26 10:02:11", "company": "苹果", "ratal": 7784.7252}
{"index":{}}
{"date": "2021-01-26 10:02:15", "company": "小米", "ratal": 185000}
{"index":{}}
{"date": "2021-01-26 10:01:23", "company": "阿里", "ratal": 1072526}
{"index":{}}
{"date": "2021-01-27 10:01:54", "company": "腾讯", "ratal": 6500}
{"index":{}}
{"date": "2021-01-28 10:01:32", "company": "蚂蚁金服", "ratal": 5000}
{"index":{}}
{"date": "2021-01-29 10:01:21", "company": "字节跳动", "ratal": 10000}
{"index":{}}
{"date": "2021-01-30 10:02:07", "company": "中国石油", "ratal": 18302097}
{"index":{}}
{"date": "1648100904", "company": "中国石化", "ratal": 32654722}
{"index":{}}
{"date": "2021-11-1 12:20:00", "company": "国家电网", "ratal": 82950000}
```

然而我们以上代码查看 tax 索引的 mapping，会惊奇的发现 date 居然是一个 text 类型。

##### 原理

原因就在于对时间类型的格式的要求是绝对严格的。要求必须是一个标准的 UTC 时间类型。上述字段的数据格式如果想要使用，就必须使用`yyyy-MM-ddTHH:mm:ssZ`格式（其中T个间隔符，Z代表 0 时区），以下均为错误的时间格式（均无法被自动映射器识别为日期时间类型）：

- yyyy-MM-dd HH:mm:ss
- yyyy-MM-dd
- 时间戳

注意：需要注意的是时间说是必须的时间格式，但是需要通过手工映射方式在索引创建之前指定为日期类型，使用自动映射器无法映射为日期类型。

##### 采用手工映射

如果我们换一个思路，使用手工映射提前指定日期类型，那会又是一个什么结果呢？

```json
PUT tax
{
  "mappings": {
    "properties": {
      "date": {
        "type": "date"
      }
    }
  }
}
POST tax/_bulk
{"index":{}}
{"date": "2021-01-30 10:02:07", "company": "中国石油", "ratal": 18302097}
{"index":{}}
{"date": "1648100904", "company": "中国石化", "ratal": 32654722}
{"index":{}}
{"date": "2021-11-1T12:20:00Z", "company": "国家电网", "ratal": 82950000}
{"index":{}}
{"date": "2021-01-30T10:02:07Z", "company": "中国石油", "ratal": 18302097}
{"index":{}}
{"date": "2021-01-25", "company": "中国烟草", "ratal": 5700000}
```

执行以上代码，以下为完整的执行结果：

```json
{
  "took" : 17,
  "errors" : true,
  "items" : [
    {
      "index" : {
        "_index" : "tax",
        "_type" : "_doc",
        "_id" : "f4uyun8B1ovRQq6Sn9Qg",
        "status" : 400,
        "error" : {
          "type" : "mapper_parsing_exception",
          "reason" : "failed to parse field [date] of type [date] in document with id 'f4uyun8B1ovRQq6Sn9Qg'. Preview of field's value: '2021-01-30 10:02:07'",
          "caused_by" : {
            "type" : "illegal_argument_exception",
            "reason" : "failed to parse date field [2021-01-30 10:02:07] with format [strict_date_optional_time||epoch_millis]",
            "caused_by" : {
              "type" : "date_time_parse_exception",
              "reason" : "date_time_parse_exception: Failed to parse with all enclosed parsers"
            }
          }
        }
      }
    },
    {
      "index" : {
        "_index" : "tax",
        "_type" : "_doc",
        "_id" : "gIuyun8B1ovRQq6Sn9Qg",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 3,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "tax",
        "_type" : "_doc",
        "_id" : "gYuyun8B1ovRQq6Sn9Qg",
        "status" : 400,
        "error" : {
          "type" : "mapper_parsing_exception",
          "reason" : "failed to parse field [date] of type [date] in document with id 'gYuyun8B1ovRQq6Sn9Qg'. Preview of field's value: '2021-11-1T12:20:00Z'",
          "caused_by" : {
            "type" : "illegal_argument_exception",
            "reason" : "failed to parse date field [2021-11-1T12:20:00Z] with format [strict_date_optional_time||epoch_millis]",
            "caused_by" : {
              "type" : "date_time_parse_exception",
              "reason" : "date_time_parse_exception: Failed to parse with all enclosed parsers"
            }
          }
        }
      }
    },
    {
      "index" : {
        "_index" : "tax",
        "_type" : "_doc",
        "_id" : "gouyun8B1ovRQq6Sn9Qg",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 4,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "tax",
        "_type" : "_doc",
        "_id" : "g4uyun8B1ovRQq6Sn9Qg",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 5,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}
```

分析结果：

- **第一个（写入失败）：2021-01-30 10:02:07**
- **第二个（写入成功）：1648100904**
- **第三个（写入失败）：2021-11-1T12:20:00Z**
- **第四个（写入成功）：2021-01-30T10:02:07Z**
- **第五个（写入成功）：2021-01-25**

##### 总结

- 对于`yyyy-MM-dd HH:mm:ss`或`2021-11-1T12:20:00Z`，ES 的自动映射器完全无法识别，即便是事先声明日期类型，数据强行写入也会失败。
- 对于时间戳和`yyyy-MM-dd`这样的时间格式，ES 自动映射器无法识别，但是如果事先说明了日期类型是可以正常写入的。
- 对于标准的日期时间类型是可以正常自动识别为日期类型，并且也可以通过手工映射来实现声明字段类型。

##### 解决方案

其实解决办法非常简单。只需要在字段属性中添加一个参数：“format”: “yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis”，这样就可以避免因为数据格式不统一而导致数据无法写入的窘境。

代码如下：

```json
PUT test_index
{
  "mappings": {
    "properties": {
      "time": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }
    }
  }
}
```