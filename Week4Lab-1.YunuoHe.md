# INFO300.LabExercise 4 Part One
Date: May 29, 2026

Student Name: YunuoHe
Student ID: 320240942751
Email:heyn2024@lzu.edu.cn

Goals: Practice with ElasticSearch single document APIs

+ Submit a Makedown file named "Week4Lab-1.yourName.md" and a PDF file named "Week4Lab-1.yourName.pdf" 
 (you may use this file as a template). 

Notes:
+ Use your name to replace "studentName".
+ Run the following command and write the response of each command.
+ Answer the questions.

## Working with ElasticSearch single document APIs

###  1). POST vs. PUT

Run the Command:
```json
POST /sid-学号-docs/_doc
{
  "t1":"abc"
}
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs",
  "_type" : "_doc",
  "_id" : "3hqacZ4Be3FvMxguEVY_",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

```

Run the Command:
```json
PUT /sid-学号-docs/_doc
{
  "t1":"abc"
}
```
+ Why there is an error?
+ Your Answer:
```text
PUT必须指定文档id
```
Run the Command:
```json
PUT /sid-学号-docs/_doc/1
{
  "t1":"abc"
}
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}

```
+ Result Explanation:
```text
这次的有文档id:1,所以能提交成功
```
Run the Command:
```json
PUT /sid-学号-docs/_create/2
{
  "t2":"nbc"
}
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}

```
+ Result Explanation:
```text
这次在sid-320240942751-docs下面创建一个id为2的文档，并且添加一个新的字段t2
```
Run the Command:
```json
POST /sid-学号-docs/_create/3
{
  "t2":"nba"
}
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs",
  "_type" : "_doc",
  "_id" : "3",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}

```
+ Result Explanation:
```text
使用POST方法带上文档id的方式，这时候跟PUT方法一样，因为指定了文档id
```
### 2). GET and HEAD


Run the Command:
```json
GET /sid-学号-docs/_doc/2
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 1,
  "_seq_no" : 2,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "t2" : "nbc"
  }
}

```
+ Result Explanation:
```text
使用GET获取2号文档的内容
```
Run the Command:
```json
HEAD /sid-学号-docs/_doc/2
```
Command response:
```json
200 - OK
```
+ Result Explanation:
```text
检查2号文档存在
```

Run the Command:
```json
GET /sid-学号-docs/_source/2
```
Command response:
```json
{
  "t2" : "nbc"
}

```
+ Result Explanation:
```text
source只返回原始JSON内容，不返回元数据
```
Run the Command:
```json
HEAD /sid-学号-docs/_source/1234
```
Command response:
```json
{"statusCode":404,"error":"Not Found","message":"404 - Not Found"}
```
+ What's the meaning of 404?
+ Your Answer:
```text
因为没有添加1234号文档
```

### 3). Delete

Run the Command:
```json
DELETE /sid-学号-docs/_doc/2 
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 2,
  "result" : "deleted",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 4,
  "_primary_term" : 1
}

```
+ Result Explanation:
```text
删除2号文档
```
Run the Command:
```json
POST /sid-学号-docs/_delete_by_query
{
  "query":{
    "match":{
      "t2":"nbc"
    }
  }
}
```
Command response:
```json
{
  "took" : 521,
  "timed_out" : false,
  "total" : 0,
  "deleted" : 0,
  "batches" : 0,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}

```
+ Result Explanation:
```text
删除所有t2字段是nbc的记录
```
Run the Commands:
```json
PUT /sid-学号-docs1/_doc/1
{
  "title": "Catch me if you can"
}

POST sid-学号-docs,sid-学号-docs1/_delete_by_query
{
  "query":{
    "match_all":{}
  }
}
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs1",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

{
  "took" : 28,
  "timed_out" : false,
  "total" : 4,
  "deleted" : 4,
  "batches" : 1,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}

```
+ Are there any documents in the index "sid-学号-docs1" now? Does the index "sid-学号-docs1" still exist?
+ Your Answer:
```text
文档已经删除，但是索引还在
```

### 4). Update

Run the Command:
```json
PUT /sid-学号-docs1/_doc/1
{
  "Ntest": 1,
  "date": ["2021-11-04"]
}

POST /sid-学号-docs1/_update/1
{
  "script":{
    "source": "ctx._source.Ntest+=params.aa",
    "lang": "painless", 
    "params": {
      "aa": 6
    }
  }
}
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs1",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}

{
  "_index" : "sid-320240942751-docs1",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}
```
+ What's the value of "Ntest" now?
+ Your Answer:
```text
7
```

Run the Command:
```json
POST /sid-学号-docs1/_update/1
{
  "script":{
    "source": "ctx._source.date.add(params.dt)",
    "lang": "painless", 
    "params":{
      "dt": "2024-10-28"
    }
  }
}
```
Command response:
```json
{
  "_index" : "sid-320240942751-docs1",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 4,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 5,
  "_primary_term" : 1
}

```
Write a query to see the document content of index "sid-学号-docs1"

Your Command:
```json
GET /sid-320240942751-docs1/_source/1
```
Command response:
```json
{
  "Ntest" : 7,
  "date" : [
    "2021-11-04",
    "2024-10-28",
  ]
}

```


## Working with ElasticSearch multiple document APIs

### 1). Mget

Run the Command:
```json
GET news/_mget
{
  "docs":[
    {
      "_id":"DcR6gZYByMkE0r5vw9eW"
    },
    {
      "_id":"D8R6gZYByMkE0r5vw9eW"
    }
    ]
}
```
Command response:
```json
{
  "docs" : [
    {
      "_index" : "news",
      "_type" : "_doc",
      "_id" : "DcR6gZYByMkE0r5vw9eW",
      "found" : false
    },
    {
      "_index" : "news",
      "_type" : "_doc",
      "_id" : "D8R6gZYByMkE0r5vw9eW",
      "found" : false
    }
  ]
}

```
+ Result Explanation:
```text
批量获取两个文档的内容
```
Run the Command:
```json
GET /_mget
   {
     "docs":[
       {
         "_index":"news",
         "_id":"EMR6gZYByMkE0r5vw9eW",
         "_source":["authors","headline","short_description"]
       },
       {
         "_index":"sid-320240942751-test3",
         "_id": "4"
         
       }
     ]
   }
```
Command response:
```json
{
   "docs" : [
     {
       "_index" : "news",
       "_type" : "_doc",
       "_id" : "EMR6gZYByMkE0r5vw9eW",
       "found" : false
     },
     {
       "_index" : "sid-320240942751-test3",
       "_type" : "_doc",
       "_id" : "4",
       "_version" : 1,
       "_seq_no" : 1,
       "_primary_term" : 1,
       "found" : true,
       "_source" : {
         "name" : "mike",
         "age" : 23
       }
     }
   ]
 }
 

```
+ Result Explanation:
```text
第一个文档索引存在，但是文档不存在。第二个是索引与文档都存在
```
### 2). Bulk API
Run the Command:
```json
POST /_bulk
{"index":{"_index":"sid-学号-test3","_id":"3"}}
{"name":"tom","age":21}
{"update":{"_index":"sid-学号-test3","_id":"1"}}
{"doc":{"t2":"male"}}
{"index":{"_index":"sid-学号-test3","_id":"4"}}
{"name":"mike","age":23}
```
Command response:
```json
{
  "took" : 641,
  "errors" : true,
  "items" : [
    {
      "index" : {
        "_index" : "sid-320240942751-test3",
        "_type" : "_doc",
        "_id" : "3",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 0,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "update" : {
        "_index" : "sid-320240942751-test3",
        "_type" : "_doc",
        "_id" : "1",
        "status" : 404,
        "error" : {
          "type" : "document_missing_exception",
          "reason" : "[_doc][1]: document missing",
          "index_uuid" : "Mkf0PIzBQr-nAexWu5n4_g",
          "shard" : "0",
          "index" : "sid-320240942751-test3"
        }
      }
    },
    {
      "index" : {
        "_index" : "sid-320240942751-test3",
        "_type" : "_doc",
        "_id" : "4",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}

```
+ Result Explanation:
```text
三个请求批量操作，创建索引文档，更新文档内容，创建索引文档
```
### 3). Reindex API
Run the Command:
```json
POST /_reindex
{
  "source":{
    "index":"sid-学号-test3"
  },
  "dest": {
    "index":"sid-学号-test4"
  }
}
```
Command response:
```json
{
  "took" : 773,
  "timed_out" : false,
  "total" : 2,
  "updated" : 0,
  "created" : 2,
  "deleted" : 0,
  "batches" : 1,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}

```
+ Result Explanation:
```text
将索引 sid-320240942751-test3 中的所有文档复制到 sid-320240942751-test4 中。
```