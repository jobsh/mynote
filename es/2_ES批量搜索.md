# 批量搜索

* 通过ids实现批量搜索

  ```JSON
  GET		http://192.168.248.160:9200/shop/_doc/_search
  
  {
  	"query": {
  		"ids": {
  			"type": "_doc",
  			"values": ["1001", "1003", "1005"]
  		}
      }
  }
  ```

* mget

  ```
  GET		http://192.168.248.160:9200/shop/_doc/_mget
  ```

  ```jsons
  {
  	"ids":  ["1001", "1003", "1005"]
  }
  ```

  

# **批量操作 bulk**

## 前期准备

* 创建shop2 索引

  ```
  PUT	http://192.168.248.160:9200/shop2/
  ```

  

### 基本语法

bulk操作和以往的普通请求格式有区别。不要格式化json，不然就不在同一行了，这个需要注意。

```
{ action: { metadata }}\n
{ request body        }\n
{ action: { metadata }}\n
{ request body        }\n
...
```

- `{ action: { metadata }}`代表批量操作的类型，可以是新增、删除或修改
- `\n`是每行结尾必须填写的一个规范，每一行包括最后一行都要写，用于es的解析
- `{ request body }`是请求body，增加和修改操作需要，删除操作则不需要

### 批量操作的类型

action 必须是以下选项之一:

- create：如果文档不存在，那么就创建它。存在会报错。发生异常报错不会影响其他操作。
- index：创建一个新文档或者替换一个现有的文档。
- update：部分更新一个文档。
- delete：删除一个文档。

metadata 中需要指定要操作的文档的`_index 、 _type 和 _id`，`_index 、 _type`也可以在url中指定

### 实操

- create新增文档数据，在metadata中指定index以及type

  ```
  POST    /_bulk
  {"create": {"_index": "shop2", "_type": "_doc", "_id": "2001"}}
  {"id": "2001", "nickname": "name2001"}
  {"create": {"_index": "shop2", "_type": "_doc", "_id": "2002"}}
  {"id": "2002", "nickname": "name2002"}
  {"create": {"_index": "shop2", "_type": "_doc", "_id": "2003"}}
  {"id": "2003", "nickname": "name2003"}
  ```

- create创建已有id文档，在url中指定index和type

  ```
  POST    /shop/_doc/_bulk
  {"create": {"_id": "2003"}}
  {"id": "2003", "nickname": "name2003"}
  {"create": {"_id": "2004"}}
  {"id": "2004", "nickname": "name2004"}
  {"create": {"_id": "2005"}}
  {"id": "2005", "nickname": "name2005"}
  ```

- index创建，已有文档id会被覆盖，不存在的id则新增

  ```
  POST    /shop/_doc/_bulk
  {"index": {"_id": "2004"}}
  {"id": "2004", "nickname": "index2004"}
  {"index": {"_id": "2007"}}
  {"id": "2007", "nickname": "name2007"}
  {"index": {"_id": "2008"}}
  {"id": "2008", "nickname": "name2008"}
  ```

- update跟新部分文档数据

```
POST    /shop/_doc/_bulk
{"update": {"_id": "2004"}}
{"doc":{ "id": "3004"}}
{"update": {"_id": "2007"}}
{"doc":{ "nickname": "nameupdate"}}
```

- delete批量删除

```
POST    /shop/_doc/_bulk
{"delete": {"_id": "2004"}}
{"delete": {"_id": "2007"}}

```

- 综合批量各种操作

  ```
  POST    /shop/_doc/_bulk
  {"create": {"_id": "8001"}}
  {"id": "8001", "nickname": "name8001"}
  {"update": {"_id": "2001"}}
  {"doc":{ "id": "20010"}}
  {"delete": {"_id": "2003"}}
  {"delete": {"_id": "2005"}}
  ```

- 官文：<https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html>

**注意：** 只能根据 _id 删除，不能根据自己定义的field删除，即使自己定义的field的value是全局唯一的，也会抛出异常。

所有的{ action: { metadata }}\n 只能选择 **_id** 进行操作。