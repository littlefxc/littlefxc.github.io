---
title: Elasticsearch 的 REST API 的使用介绍
date: 2021-01-09 16:41:04
categories: ELK
tags: es
---

[TOC]

# 1 集群健康

[集群健康 | Elasticsearch: 权威指南 | Elastic](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_cluster_health.html)

```
GET /_cluster/health
```

# 2 索引相关

## 2.1 创建索引

```
PUT /index_test
{
    "settings": {
        "index": {
            "number_of_shards": "2",
            "number_of_replicas": "0"
        }
    }
}
```

- number_of_shards : 分片数
- number_of_replicas : 副本数


##  2.2 查看索引

```
GET _cat/indices?v
```

## 2.3 删除索引

```
DELETE /index_test
```

# 3 mappings 相关

## 3.1 创建索引的同时创建mappings

```
PUT /index_test
{
    "settings": {
        "index": {
            "number_of_shards": "3",
            "number_of_replicas": "0"
        }
    },
    "mappings": {
        "properties": {
            "realname": {
            	"type": "text",
            	"index": true
            },
            "username": {
            	"type": "keyword",
            	"index": false
            },
            "id": {
        	    "type": "long"
            },
            "age": {
            	"type": "integer"
            },
            "nickname": {
                "type": "keyword"
            },
            "money1": {
                "type": "float"
            },
            "money2": {
                "type": "double"
            },
            "sex": {
                "type": "byte"
            },
            "score": {
                "type": "short"
            },
            "is_teenager": {
                "type": "boolean"
            },
            "birthday": {
                "type": "date"
            },
            "relationship": {
                "type": "object"
            }
        }
    }
}
```

注意：

- **number_of_shards** : 分片数

- **number_of_replicas** : 副本数

- **index**：默认true，设置为false的话，那么这个字段就不会被索引(例如密码等敏感信息)

- 某个属性一旦被建立，就不能修改了，但是可以新增额外属性

- 主要数据类型：
  - text, keyword, ~~string~~
  - long, integer, short, byte
  - double, float
  - boolean
  - date
  - object
  - 数组不能混，类型一致

- **text**：文字类需要被分词被倒排索引的内容，比如`商品名称`，`商品详情`，`商品介绍`，使用text。

- **keyword**：不会被分词，不会被倒排索引，直接匹配搜索，比如`订单状态`，`用户qq`，`微信号`，`手机号`等，这些精确匹配，无需分词。

# 4 文档的基本操作

## 4.1 添加文档

```
POST /{索引名}/_doc/{索引ID}（是指索引在es中的id，而不是这条记录的id，比如记录的id从数据库来是1001，并不是这个。如果不写，则自动生成一个字符串。建议和数据id保持一致> ）

POST /user/_doc/1
{
    "id": 1,
    "username": "username1",
    "password": "password1",
    "create_date": "2021-01-09"
}
```

**查看索引**

```
GET /user
```

**结果如下**

```
{
  "user" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "create_date" : {
          "type" : "date"
        },
        "id" : {
          "type" : "long"
        },
        "password" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "username" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1610183188727",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "55w3gLZNTVyJnmdi6hTo5g",
        "version" : {
          "created" : "7100099"
        },
        "provided_name" : "user"
      }
    }
  }
}
```

**注意**：

- 如果索引没有手动建立mappings，那么当插入文档数据的时候，会根据文档类型自动设置属性类型。这个就是es的动态映射，帮我们在index索引库中去建立数据结构的相关配置信息。
- “fields”: {“type”: “keyword”}对一个字段设置多种索引模式，使用text类型做全文检索，也可使用keyword类型做聚合和排序
- “ignore_above” : 256设置字段索引和存储的长度最大值，超过则被忽略

## 4.2 修改文档

###  4.2.1 局部修改

```
POST /user/_doc/1/_update
{
    "doc": {
        "name": "update1"
    }
}
```

### 4.2.2 全局修改

```
PUT /user/_doc/1
{
    "id": 1,
    "username": "update2",
    "password": "password1",
    "create_date": "2021-01-09"
}
```

同时每次修改后，返回参数中的属性 `verison` 都会更改

## 4.3 删除文档

```
DELETE /user/_doc/1
```

**注**：文档删除不是立即删除，文档还是保存在磁盘上，索引增长越来越多，才会把那些曾经标识过删除的，进行清理，从磁盘上移出去。

## 4.4 查询文档

常规查询

```bash
GET /index_demo/_doc/1
GET /index_demo/_doc/_search
```

### 4.4.1 查询结果

```bash
{
    "_index": "my_doc",
    "_type": "_doc",
    "_id": "2",
    "_score": 1.0,
    "_version": 9,
    "_source": {
        "id": 1002,
        "name": "imooc-2",
        "desc": "imooc is fashion",
        "create_date": "2019-12-25"
    }
}
```

### 4.4.2 元数据

- _index：文档数据所属那个索引，理解为数据库的某张表即可。
- _type：文档数据属于哪个类型，新版本使用`_doc` 。
- _id：文档数据的唯一标识，类似数据库中某张表的主键。可以自动生成或者手动指定。
- _score：查询相关度，是否契合用户匹配，分数越高用户的搜索体验越高。
- _version：版本号。
- _source：文档数据，json格式。

### 4.4.3 定制结果集

```bash
GET /index_demo/_doc/1?_source=id,name
GET /index_demo/_doc/_search?_source=id,name
```

### 4.4.4 判断文档是否存在

```bash
HEAD /index_demo/_doc/1
```