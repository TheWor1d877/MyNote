# INFO300 2026 Spring Term -- Lab Exercises 2

Goals: Learn to start working with ElasticSearch and be able to run simple 
queries and add documents to ElasticSearch.

Grading:  You name must become searchable in the collection "info3002026springstudents".

Instructions: Please use the provided **lab2.md** template and follow the
steps below to prepare your answers in the **Markdown** format.
You cannot change anything in this file, except in between the following tags, where you should include your answers.
Thus, for each question, replace YOUR ANSWER HERE by your answer and keep everything else untouched.
#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer


## 1. Working with ElasticSearch through Kibana

+ Logon to Kibana and open the "Dev Tools"
+ Run the following query:
```
GET /info3002026springstudents/_search
{
  "query": {
    "match_all": {}
  }
}
```
+ Observe the query results on the right.
+ Currently, how many results are returned from this query (Hits total value):

#### Beginning of Your Answer
```
2
```
#### End of Your Answer

## 2. Adding your name to the collection by running a similar query like this:
```
POST /info3002026springstudents/_doc/
{
  "studentID": "2020030405",
  "name":"Jielun Zhou",
  "class": "Data Science 2025",
  "from": "Fujian",
  "favorite_food":"Orange Beef",
  "favorite_fruit":"Apple",
  "favorite_university":"LZU",
  "interests":["reading", "traveling", "programming", "information retrieval", "visualization", "information visualization"]
}
```
Requirements:
+ If you do it successfully, your name should become searchable.
+ You should enter one item each for your "favorite_food", "favorite_fruit", and favor "favorite_university".
+ Enter at least three items in the "interests" array.

Provide the query you executed next:

#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer

Also provide the **response/results**

#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer

## 3. Running a similar query to verify your name is there:
```
GET /info3002026springstudents/_search
{
  "query": {
    "match": {
      "name": "Jielun Zhou"
    }
  }
}
```

Provide the query you executed next:

#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer

Also provide the **response/results**

#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer


## 4. Adding another document using "PUT":
```
PUT /info3002026springstudents/_doc/jzhou101
{
  "studentID": "jzhou101",
  "name":"周杰伦",
  "class": "Lanzhou2019",
  "from": "Fujian",
  "favorite_food":"Beef",
  "favorite_fruit":"Orange",
  "favorite_university":"Lanzhou University",
  "interest":["reading", "traveling", "programming", "information retrieval", "visualization", "information visualization"]
}
```
Requirements:
+ Make sure to use your studentID as the documentID on the command!
+ Enter your Chinese name this time.
+ You should modify some of your favorite items there.
+ Run a query to make sure your entry is there.

Provide the query you executed next:

#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer


## 5. Querying:

Run a query similar to the following one so you can verify if both of your names are in the collection 
(this is equivalent to the boolean query ("Jielun Zhou" **AND** "周杰伦")):

```
get /info3002026springstudents/_search
{
  "query": {
  "bool": {
    "should": [
      {"match": {"name": "Jielun Zhou"}},
      {"match": {"name": "周杰伦"}}
    ]
  }
  }
}
```
Provide the query you executed next:

#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer

Also provide the **response/results**

#### Beginning of Your Answer
```
YOUR ANSWER HERE
```
#### End of Your Answer


## 6. Now you have completed the first try of using ElasticSearch. You can also try the examples from the lectures.

## 7. Explore the documentation of ElasticSearch to see what other queries you can run on this collection.

  + Introduction to ElasticSearch:
https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html 



## Assignment Submission

Please submit all your answers a the Markdown file. Make sure:

1.  File name has to be **lab2_YourStudentID.md**.

2.  All the answers must be in the corresponding place for them. 

3. Submit only the file **lab2_YourStudentID.md**.
