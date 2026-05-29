# INFO300.LabExercise 4 Part Two
Date: May 29, 2026

Student:Yunuo He
Email: heyn2024@lzu.edu.cn
Student ID:320240942751

+ Goals: Practice with ElasticSearch Analyzer 


+ Submit a Makedown and a PDF file named "Week4Lab-2.yourName" 
 (you may use this file as a template ) 


### 1). Given the following text, use "whitespace", "standard", and "english" Analyzer to analyze the text.

+ text: "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01."

+ command example:
```json
POST _analyze
{
  "analyzer":"",
  "text": ""
}
```
Your Command 1:
```json
POST _analyze
{
  "analyzer": "whitespace",
  "text": "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01."
}
```

Your Command 2:
```json
POST _analyze
{
  "analyzer": "standard",
  "text": "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01."
}
```

Your Command 3:
```json
POST _analyze
{
  "analyzer": "english",
  "text": "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01."
}
```

Write the tokens under the three analyzers (sperated by semicolon):

+ whitespace:
```text
{
  "tokens" : [
    {
      "token" : "Lanzhou",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "University",
      "start_offset" : 8,
      "end_offset" : 18,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "has",
      "start_offset" : 19,
      "end_offset" : 22,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "2,135",
      "start_offset" : 23,
      "end_offset" : 28,
      "type" : "word",
      "position" : 3
    },
    {
      "token" : "full-time",
      "start_offset" : 29,
      "end_offset" : 38,
      "type" : "word",
      "position" : 4
    },
    {
      "token" : "teaching",
      "start_offset" : 39,
      "end_offset" : 47,
      "type" : "word",
      "position" : 5
    },
    {
      "token" : "faculty,",
      "start_offset" : 48,
      "end_offset" : 56,
      "type" : "word",
      "position" : 6
    },
    {
      "token" : "1,695",
      "start_offset" : 57,
      "end_offset" : 62,
      "type" : "word",
      "position" : 7
    },
    {
      "token" : "graduate",
      "start_offset" : 63,
      "end_offset" : 71,
      "type" : "word",
      "position" : 8
    },
    {
      "token" : "students’",
      "start_offset" : 72,
      "end_offset" : 81,
      "type" : "word",
      "position" : 9
    },
    {
      "token" : "supervisors",
      "start_offset" : 82,
      "end_offset" : 93,
      "type" : "word",
      "position" : 10
    },
    {
      "token" : "and",
      "start_offset" : 94,
      "end_offset" : 97,
      "type" : "word",
      "position" : 11
    },
    {
      "token" : "588",
      "start_offset" : 98,
      "end_offset" : 101,
      "type" : "word",
      "position" : 12
    },
    {
      "token" : "professors",
      "start_offset" : 102,
      "end_offset" : 112,
      "type" : "word",
      "position" : 13
    },
    {
      "token" : "on",
      "start_offset" : 113,
      "end_offset" : 115,
      "type" : "word",
      "position" : 14
    },
    {
      "token" : "2024/10/01.",
      "start_offset" : 116,
      "end_offset" : 127,
      "type" : "word",
      "position" : 15
    }
  ]
}

```

+ standard: 
```text
{
  "tokens" : [
    {
      "token" : "lanzhou",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "university",
      "start_offset" : 8,
      "end_offset" : 18,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "has",
      "start_offset" : 19,
      "end_offset" : 22,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "2,135",
      "start_offset" : 23,
      "end_offset" : 28,
      "type" : "<NUM>",
      "position" : 3
    },
    {
      "token" : "full",
      "start_offset" : 29,
      "end_offset" : 33,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "time",
      "start_offset" : 34,
      "end_offset" : 38,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "teaching",
      "start_offset" : 39,
      "end_offset" : 47,
      "type" : "<ALPHANUM>",
      "position" : 6
    },
    {
      "token" : "faculty",
      "start_offset" : 48,
      "end_offset" : 55,
      "type" : "<ALPHANUM>",
      "position" : 7
    },
    {
      "token" : "1,695",
      "start_offset" : 57,
      "end_offset" : 62,
      "type" : "<NUM>",
      "position" : 8
    },
    {
      "token" : "graduate",
      "start_offset" : 63,
      "end_offset" : 71,
      "type" : "<ALPHANUM>",
      "position" : 9
    },
    {
      "token" : "students",
      "start_offset" : 72,
      "end_offset" : 80,
      "type" : "<ALPHANUM>",
      "position" : 10
    },
    {
      "token" : "supervisors",
      "start_offset" : 82,
      "end_offset" : 93,
      "type" : "<ALPHANUM>",
      "position" : 11
    },
    {
      "token" : "and",
      "start_offset" : 94,
      "end_offset" : 97,
      "type" : "<ALPHANUM>",
      "position" : 12
    },
    {
      "token" : "588",
      "start_offset" : 98,
      "end_offset" : 101,
      "type" : "<NUM>",
      "position" : 13
    },
    {
      "token" : "professors",
      "start_offset" : 102,
      "end_offset" : 112,
      "type" : "<ALPHANUM>",
      "position" : 14
    },
    {
      "token" : "on",
      "start_offset" : 113,
      "end_offset" : 115,
      "type" : "<ALPHANUM>",
      "position" : 15
    },
    {
      "token" : "2024",
      "start_offset" : 116,
      "end_offset" : 120,
      "type" : "<NUM>",
      "position" : 16
    },
    {
      "token" : "10",
      "start_offset" : 121,
      "end_offset" : 123,
      "type" : "<NUM>",
      "position" : 17
    },
    {
      "token" : "01",
      "start_offset" : 124,
      "end_offset" : 126,
      "type" : "<NUM>",
      "position" : 18
    }
  ]
}

```
+ english: 
```text
{
  "tokens" : [
    {
      "token" : "lanzhou",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "univers",
      "start_offset" : 8,
      "end_offset" : 18,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "ha",
      "start_offset" : 19,
      "end_offset" : 22,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "2,135",
      "start_offset" : 23,
      "end_offset" : 28,
      "type" : "<NUM>",
      "position" : 3
    },
    {
      "token" : "full",
      "start_offset" : 29,
      "end_offset" : 33,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "time",
      "start_offset" : 34,
      "end_offset" : 38,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "teach",
      "start_offset" : 39,
      "end_offset" : 47,
      "type" : "<ALPHANUM>",
      "position" : 6
    },
    {
      "token" : "faculti",
      "start_offset" : 48,
      "end_offset" : 55,
      "type" : "<ALPHANUM>",
      "position" : 7
    },
    {
      "token" : "1,695",
      "start_offset" : 57,
      "end_offset" : 62,
      "type" : "<NUM>",
      "position" : 8
    },
    {
      "token" : "graduat",
      "start_offset" : 63,
      "end_offset" : 71,
      "type" : "<ALPHANUM>",
      "position" : 9
    },
    {
      "token" : "student",
      "start_offset" : 72,
      "end_offset" : 80,
      "type" : "<ALPHANUM>",
      "position" : 10
    },
    {
      "token" : "supervisor",
      "start_offset" : 82,
      "end_offset" : 93,
      "type" : "<ALPHANUM>",
      "position" : 11
    },
    {
      "token" : "588",
      "start_offset" : 98,
      "end_offset" : 101,
      "type" : "<NUM>",
      "position" : 13
    },
    {
      "token" : "professor",
      "start_offset" : 102,
      "end_offset" : 112,
      "type" : "<ALPHANUM>",
      "position" : 14
    },
    {
      "token" : "2024",
      "start_offset" : 116,
      "end_offset" : 120,
      "type" : "<NUM>",
      "position" : 16
    },
    {
      "token" : "10",
      "start_offset" : 121,
      "end_offset" : 123,
      "type" : "<NUM>",
      "position" : 17
    },
    {
      "token" : "01",
      "start_offset" : 124,
      "end_offset" : 126,
      "type" : "<NUM>",
      "position" : 18
    }
  ]
}

```

### 2). Create an index of "sid-学号-books" and customized Analyzer with the type of "standard", the max token length of "9", the stopword of "_english_".

Command example:
```json
PUT /suwei_books
{
 "settings": {
   "analysis": {
     "analyzer": {
       "sid-学号-analyzer1":{
         "type":"",
         "max_token_length":"",
         "stopwords":""
       }
     }
   }
 }
}
```

Your Command 4:
```json
PUT /sid-320240942751-books
{
  "settings": {
    "analysis": {
      "analyzer": {
        "sid-320240942751-analyzer1": {
          "type": "standard",
          "max_token_length": 9,
          "stopwords": "_english_"
        }
      }
    }
  }
}
```

Use the analyzer `sid-学号-analyzer1` to analyze the text: "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01.".

Command example:
```json
POST /suwei_books/_analyze
{
  "analyzer":"",
  "text": ""
}
```

Your Command 5:
```json
POST /sid-320240942751-books/_analyze
{
  "analyzer": "sid-320240942751-analyzer1",
  "text": "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01."
}
```

Write all the tokens under the new Analyzer (sperated by semicolon):

+ `sid-学号-analyzer1`:
```text
{
  "tokens" : [
    {
      "token" : "lanzhou",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "univers",
      "start_offset" : 8,
      "end_offset" : 18,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "ha",
      "start_offset" : 19,
      "end_offset" : 22,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "2,135",
      "start_offset" : 23,
      "end_offset" : 28,
      "type" : "<NUM>",
      "position" : 3
    },
    {
      "token" : "full",
      "start_offset" : 29,
      "end_offset" : 33,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "time",
      "start_offset" : 34,
      "end_offset" : 38,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "teach",
      "start_offset" : 39,
      "end_offset" : 47,
      "type" : "<ALPHANUM>",
      "position" : 6
    },
    {
      "token" : "faculti",
      "start_offset" : 48,
      "end_offset" : 55,
      "type" : "<ALPHANUM>",
      "position" : 7
    },
    {
      "token" : "1,695",
      "start_offset" : 57,
      "end_offset" : 62,
      "type" : "<NUM>",
      "position" : 8
    },
    {
      "token" : "graduat",
      "start_offset" : 63,
      "end_offset" : 71,
      "type" : "<ALPHANUM>",
      "position" : 9
    },
    {
      "token" : "student",
      "start_offset" : 72,
      "end_offset" : 80,
      "type" : "<ALPHANUM>",
      "position" : 10
    },
    {
      "token" : "supervisor",
      "start_offset" : 82,
      "end_offset" : 93,
      "type" : "<ALPHANUM>",
      "position" : 11
    },
    {
      "token" : "588",
      "start_offset" : 98,
      "end_offset" : 101,
      "type" : "<NUM>",
      "position" : 13
    },
    {
      "token" : "professor",
      "start_offset" : 102,
      "end_offset" : 112,
      "type" : "<ALPHANUM>",
      "position" : 14
    },
    {
      "token" : "2024",
      "start_offset" : 116,
      "end_offset" : 120,
      "type" : "<NUM>",
      "position" : 16
    },
    {
      "token" : "10",
      "start_offset" : 121,
      "end_offset" : 123,
      "type" : "<NUM>",
      "position" : 17
    },
    {
      "token" : "01",
      "start_offset" : 124,
      "end_offset" : 126,
      "type" : "<NUM>",
      "position" : 18
    }
  ]
}
```

### 3). Create a new index with name "sid-学号-analyzertest". Set the type of the field "introduction" to "text". Set the search time analyzer to "english" and the index time analyzer to "standard".

Command Example:
```json
PUT /suwei_analyzertest
{
  "mappings": {
   "properties": {
     "FieldName": {
       "type": "",
       "analyzer": "",
       "search_analyzer": ""
      }
    }
 }
}
```

Your Command 6:
```json

PUT /sid-320240942751-analyzertest
{
  "mappings": {
    "properties": {
      "introduction": {
        "type": "text",
        "analyzer": "standard",
        "search_analyzer": "english"
      }
    }
  }
}
```

Index a document including a field "introduction" with the text: "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01.".

Command example:
```json
PUT /sid-学号-analyzertest/_doc/1
{
  "": ""
}
```

Your Command 7:
```json
PUT /sid-320240942751-analyzertest/_doc/1
{
  "introduction": "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01."
}
```

Write three queries to retrieval the following text:

+ "University"
+ "university"
+ "full time"
+ "supervisors"
+ "supervisor"
+ "and"

Your Command 7:
```json
GET /sid-320240942751-analyzertest/_search
{
  "query": {
    "match": {
      "introduction": "University"
    }
  }
}
```
Response 7:
```json
{
   "took" : 1,
   "timed_out" : false,
   "_shards" : {
     "total" : 1,
     "successful" : 1,
     "skipped" : 0,
     "failed" : 0
   },
   "hits" : {
     "total" : {
       "value" : 0,
       "relation" : "eq"
     },
     "max_score" : null,
     "hits" : [ ]
   }
 }
 
```

Your Command 8:
```json
GET /sid-320240942751-analyzertest/_search
{
  "query": {
    "match": {
      "introduction": "university"
    }
  }
}
```

Response 8:
```json
GET /sid-320240942751-analyzertest/_search
 {
   "query": {
     "match": {
       "introduction": "university"
     }
   }
 }
```

Your Command 9:
```json
GET /sid-320240942751-analyzertest/_search
{
  "query": {
    "match": {
      "introduction": "full time"
    }
  }
}
```
Response 9:
```json
{
   "took" : 1,
   "timed_out" : false,
   "_shards" : {
     "total" : 1,
     "successful" : 1,
     "skipped" : 0,
     "failed" : 0
   },
   "hits" : {
     "total" : {
       "value" : 1,
       "relation" : "eq"
     },
     "max_score" : 0.36464313,
     "hits" : [
       {
         "_index" : "sid-320240942751-analyzertest",
         "_type" : "_doc",
         "_id" : "1",
         "_score" : 0.36464313,
         "_source" : {
           "introduction" : "Lanzhou University has 2,135 full-time teaching faculty, 1,695 graduate students’ supervisors and 588 professors on 2024/10/01."
         }
       }
     ]
   }
 }
 
```

Your Command 10:
```json
GET /sid-320240942751-analyzertest/_search
{
  "query": {
    "match": {
      "introduction": "supervisors"
    }
  }
}
```
Response 10:
```json
{
   "took" : 1,
   "timed_out" : false,
   "_shards" : {
     "total" : 1,
     "successful" : 1,
     "skipped" : 0,
     "failed" : 0
   },
   "hits" : {
     "total" : {
       "value" : 0,
       "relation" : "eq"
     },
     "max_score" : null,
     "hits" : [ ]
   }
 }
 
```

Your Command 11:
```json
GET /sid-320240942751-analyzertest/_search
   {
     "query": {
       "match": {
         "introduction": "supervisor"
       }
     }
   }
```
Response 11:
```json
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```

Your Command 12:
```json
GET /sid-320240942751-analyzertest/_search
{
  "query": {
    "match": {
      "introduction": "and"
    }
  }
}
```
Response 12:
```json
{
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
    },
    "hits" : {
      "total" : {
        "value" : 0,
        "relation" : "eq"
      },
      "max_score" : null,
      "hits" : [ ]
    }
  }
```

Give a short description why there IS or IS NO hit for each query.

Your Answer:
* "University":
```text
索引时standard分析器存储"university"，查询时english分析器转为词干"univers"，两者不匹配。
```

* "university":
```text
没有命中。同上，索引存储"university"，查询转为"univers"，不匹配。
```
* "full time":
```text
命中。索引时"full-time"被拆分为"full"和"time"存储，查询时同样得到"full"和"time"，匹配成功。
```

* "supervisors":
```text
没有命中。索引存储"supervisors"，查询时english将"supervisors"转为"supervisor"，但索引中没有"supervisor"，只有"supervisors"，不匹配。
```

* "supervisor":
```text
没有命中。索引存储"supervisors"，查询时english将"supervisor"转为"supervisor"，索引中只有"supervisors"（复数），不匹配。
```

* "and":
```text
没有命中。查询时english分析器将"and"识别为停用词并丢弃，无查询词去匹配。
```
