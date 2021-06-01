# ES基本操作

## **索引的一些操作**

#### 集群健康

<https://www.elastic.co/guide/cn/elasticsearch/guide/current/_cluster_health.html#_cluster_health>

```xml
GET     /_cluster/health
```

#### 创建索引

```json
PUT     /index_test
{
    "settings": {
        "index": {
            "number_of_shards": "2",  
            "number_of_replicas": "0"
        }
    }
}
```

#### 对索引进行设置

```
PUT /testindex/_settings
{
   "number_of_replicas" : 2
}
```

#### 查看索引

```xml
	1、 查看所有的索引元信息(固定的)

	GET     _cat/indices?v

	2、 查看某个索引详细信息

	GET		http://192.168.248.160:9200/index_name
```

#### 删除索引

```xml
DELETE      /index_name
```

## 索引的mappings映射

### 0. 索引分词概念

index：默认true，设置为false的话，那么这个字段就不会被索引

引用官文：<https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-index.html>

> **index**
> The index option controls whether field values are indexed. It accepts true or false and defaults to true. Fields that are not indexed are not queryable.

#### 1. 创建索引的同时创建mappings

```xml
PUT     /index_str
{
    "mappings": {
        "properties": {
            "realname": {
            	"type": "text",
            	"index": true
            },
            "username": {
            	"type": "keyword",
            	"index": false
            }
        }
    }
}
```

#### 2.查看分词效果

```xml
GET         /index_mapping/_analyze
{
	"field": "realname",
	"text": "imooc is good"
}
```

#### 3. 尝试修改

```xml
POST        /index_str/_mapping
{
    "properties": {
        "name": {
        	   "type": "long"
        }
    }
}

无法修改，只能增加
```

#### 4. 为已存在的索引创建或创建mappings

```xml
POST        /index_str/_mapping
{
    "properties": {
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
```

- 注：某个属性一旦被建立，就不能修改了，但是可以新增额外属性

#### 主要数据类型

- text, keyword, string
- long, integer, short, byte
- double, float
- boolean
- date
- object
- 数组不能混，类型一致

### 字符串

- text：文字类需要被分词被倒排索引的内容，比如`商品名称`，`商品详情`，`商品介绍`，使用text。
- keyword：不会被分词，不会被倒排索引，直接匹配搜索，比如`订单状态`，`用户qq`，`微信号`，`手机号`等，这些精确匹配，无需分词。

## 文档操作

### 删除文档

```xml
DELETE /index_name/_doc(index_type)/1
```

- 注：文档删除不是立即删除，文档还是保存在磁盘上，索引增长越来越多，才会把那些曾经标识过删除的，进行清理，从磁盘上移出去。

### 修改文档

- 局部：

  ```json
  POST /my_doc/_doc/1/_update
  
  my_doc: 索引名
  _doc: 索引类型
  1: 文档 _id
  
  {
      "doc": {
          "name": "慕课"
      }
  }
  ```

- 全量替换：

  ```json
  PUT/POST /my_doc/_doc/1
  {
       "id": 1001,
      "name": "imooc-1",
      "desc": "imooc is very good, 慕课网非常牛！",
      "create_date": "2019-12-24"
  }
  ```

- 注：每次修改后，version会更改

### 查询文档

常规查询

```xml
GET /index_demo/_doc/1
GET /index_demo/_doc/_search
```

#### 查询结果

```xml
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

#### 元数据

- _index：文档数据所属那个索引，理解为数据库的某张表即可。
- _type：文档数据属于哪个类型，新版本使用`_doc`。
- _id：文档数据的唯一标识，类似数据库中某张表的主键。可以自动生成或者手动指定。
- _score：查询相关度，是否契合用户匹配，分数越高用户的搜索体验越高。
- _version：版本号。
- _source：文档数据，json格式。

#### 定制结果集

```xml
GET /index_demo/_doc/1?_source=id,name
GET /index_demo/_doc/_search?_source=id,name
```

#### 判断文档是否存在

```xml
HEAD /index_demo/_doc/1
```
