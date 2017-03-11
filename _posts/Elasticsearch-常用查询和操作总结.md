---
title: '[Elasticsearch] 常用查询和操作总结'
date: 2015-08-28 00:12:00
categories: ['工具, 库与框架', Elasticsearch]
tags: [Elasticsearch, 常用操作]
---

##1. 取得某个索引中某个字段中的所有出现过的值

这种操作类似于使用SQL的SELECT UNIQUE语句。当需要获取某个字段上的所有可用值时，可以使用terms聚合查询完成：

```json
GET /index_streets/_search?search_type=count
{
 "aggs": {
   "street_values": {
     "terms": {
       "field": "name.raw",
       "size": 0
     }
   }
 } 
} 
```

<!-- More -->

因为目标是得到name字段上的所有出现过的值，因此search_type被设置为了count，这样在返回的响应中不会出现冗长的hits部分。另外，查询的目标字段的索引类型需要设置为not_analyzed。所以上面的field指定的是name.raw。

得到的响应如下所示：
```json
{
   "took": 23,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 7445,
      "max_score": 0,
      "hits": []
   },
   "aggregations": {
      "street_values": {
         "doc_count_error_upper_bound": 0,
         "sum_other_doc_count": 0,
         "buckets": [
            {
               "key": "江苏路",
               "doc_count": 29
            },
            {
               "key": "南京东路",
               "doc_count": 28
            },
         ...
      ...
   ...
```

---
##2. 取得某个索引/类型下某个字段中出现的不同值的个数

这种操作类似于使用SQL的select count( * ) from (select distinct * from table)语句。当需要获取某个字段上的出现的不同值的个数时，可以使用cardinality聚合查询完成：

```json
GET /index_streets/_search?search_type=count
{
  "aggs": {
    "uniq_streets": {
      "cardinality": {
        "field": "name.raw"
      }
    }
  }
}
```

因为目标是得到name字段上的所有出现过的值，因此search_type被设置为了count，这样在返回的响应中不会出现冗长的hits部分。另外，查询的目标字段如果是字符串类型的，那么其索引类型需要设置为not_analyzed。所以上面的field指定的是name.raw。

得到的响应如下所示：

```json
{
   "took": 96,
   "timed_out": false,
   "_shards": {
      "total": 1,
      "successful": 1,
      "failed": 0
   },
   "hits": {
      "total": 4136543,
      "max_score": 0,
      "hits": []
   },
   "aggregations": {
      "uniq_streets": {
         "value": 1951
      }
   }
}
```

返回结果表示该字段出现过1951个不同的字符串。

---

本文将会不断更新。


