---
title: Elasticsearch 查询语法
date: 2019-03-31 20:20:43
tags: [Elastic]
---
> [英文文档(最新)](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html)
> 
> [中文文档(2.x)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

# **轻量检索(`URL search`)**

```text
GET /megacorp/employee/_search?q=last_name:Smith
```

参数说明[详见](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search-uri-request.html#_parameters_3)

# **查询表达式**

Query-string 搜索通过命令非常方便地进行临时性的即席搜索 ，但它有自身的局限性（参见 [*轻量* 搜索](https://www.elastic.co/guide/cn/elasticsearch/guide/current/search-lite.html)）。Elasticsearch 提供一个丰富灵活的查询语言叫做 *查询表达式* ， 它支持构建更加复杂和健壮的查询。

*领域特定语言* （DSL）， 指定了使用一个 JSON 请求。我们可以像这样重写之前的查询所有 Smith 的搜索 

```json
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```
<-- more -->
参数说明[详见](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search-request-body.html#_parameters_4)

`from`(defualt:0) & `size`(default:10) **分页**

```json
{
  "query": { "match_all": {} },
  "from": 10,
  "size": 10 
}
```

`sort`**排序**

```json
{
  "query": { "match_all": {} },
  "sort" : [
        { "post_date" : {"order" : "asc"}},
        "user",
        { "name" : "desc" },
        { "age" : "desc" },
        "_score"
    ]
}
```

排序类型（sort order）

| `asc`  | 正序  |
| ------ | --- |
| `desc` | 倒序  |

排序模式 (sort mode option)

| `min`    | 最小值  |
| -------- | ---- |
| `max`    | 最大值  |
| `sum`    | 求和   |
| `avg`    | 求平均值 |
| `median` | 中间值  |

`_soucre` **显示字段**

```json
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}
```

## **match_all 查询**

match_all 查询简单的 匹配所有文档。在没有指定查询方式时，它是默认的查询：

```json
{
  "query": { "match_all": {} }
}
```

## **match查询**

无论你在任何字段上进行的是全文搜索还是精确查询，`match` 查询是你可用的标准查询。如果你在一个全文字段上使用 `match` 查询，在执行查询前，它将用正确的分析器去分析查询字符串：

```json
{ "match": { "tweet": "About Search" }}
```

如果在一个精确值的字段上使用它， 例如数字、日期、布尔或者一个 `not_analyzed` 字符串字段，那么它将会精确匹配给定的值：

```text
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}
```

## **multi_match 查询**

`multi_match` 查询可以在多个字段上执行相同的 `match` 查询：

```json
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```

## **range查询**

`range` 查询找出那些落在指定区间内的数字或者时间

```json
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```

被允许的操作符如下：

- `gt` :大于
- `gte` :大于等于
- `lt` :小于
- `lte` :小于等于

## **term查询**

```text
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}
```

`term` 查询对于输入的文本不 *分析* ，所以它将给定的值进行精确查询。

## **terms查询**

`terms` 查询和 `term` 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件：

```json
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

和`term`查询一样，`terms`查询对于输入的文本不分析。它查询那些精确匹配的值（包括在大小写、重音、空格等方面的差异）。

## **exists 查询和 missing 查询**

`exists`查询和`missing`查询被用于查找那些指定字段中有值 (`exists`) 或无值 (`missing`) 的文档。这与SQL中的`IS_NULL`(`missing`) 和`NOT IS_NULL`(`exists`) 在本质上具有共性：

```json
{
    "exists":   {
        "field":    "title"
    }
}
```

这些查询经常用于某个字段有值的情况和某个字段缺值的情况。

# **组合查询**

## **bool查询**

这种查询将多查询组合在一起，成为用户自己想要的布尔查询。它接收以下参数：

`must` like `and`

文档*必须*匹配这些条件才能被包含进来。

`must_not` like `<>`

文档*必须不*匹配这些条件才能被包含进来。

`should` like `or`

如果满足这些语句中的任意语句，将增加`_score`，否则，无任何影响。它们主要用于修正每个文档的相关性得分。

`filter`

*必须*匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。

`由于这是我们看到的第一个包含多个查询的查询，所以有必要讨论一下相关性得分是如何组合的。每一个子查询都独自地计算文档的相关性得分。一旦他们的得分被计算出来，`bool查询就将这些得分进行合并并且返回一个代表整个布尔操作的得分。

`下面的查询用于查找title字段匹配how to make millions并且不被标识为spam的文档。那些被标识为starred或在2014之后的文档，将比另外那些文档拥有更高的排名。如果 _两者_ 都满足，那么它排名将更高：`

```json
{
  "query": {
        "bool": {
            "must":     { "match": { "title": "how to make millions" }},
            "must_not": { "match": { "tag":   "spam" }},
            "should": [
                { "match": { "tag": "starred" }},
                { "range": { "date": { "gte": "2014-01-01" }}}
            ]
        }
    }
}
```

*如果没有`must`语句，那么至少需要能够匹配其中的一条`should`语句。但，如果存在至少一条`must`语句，则对`should`语句的匹配没有要求。*

## **带filter查询**

如果我们不想因为文档的时间而影响得分，可以用`filter`语句来重写前面的例子：

```json
{
    "query":{
        "bool": {
            "must":     { "match": { "title": "how to make millions" }},
            "must_not": { "match": { "tag":   "spam" }},
            "should": [
                { "match": { "tag": "starred" }}
            ],
            "filter": {
              "range": { "date": { "gte": "2014-01-01" }} 
            }
        }
    }
}
```

通过将 range 查询移到 `filter` 语句中，我们将它转成不评分的查询，将不再影响文档的相关性排名。由于它现在是一个不评分的查询，可以使用各种对 filter 查询有效的优化手段来提升性能。

所有查询都可以借鉴这种方式。将查询移到 `bool` 查询的 `filter` 语句中，这样它就自动的转成一个不评分的 filter 了。

如果你需要通过多个不同的标准来过滤你的文档，`bool` 查询本身也可以被用做不评分的查询。简单地将它放置到 `filter` 语句中并在内部构建布尔逻辑：

```json
{
    "query":{
        "bool": {
            "must":     { "match": { "title": "how to make millions" }},
            "must_not": { "match": { "tag":   "spam" }},
            "should": [
                { "match": { "tag": "starred" }}
            ],
            "filter": {
              "bool": { 
                  "must": [
                      { "range": { "date": { "gte": "2014-01-01" }}},
                      { "range": { "price": { "lte": 29.99 }}}
                  ],
                  "must_not": [
                      { "term": { "category": "ebooks" }}
                  ]
              }
            }
        }
    }
}
```

# **聚合**

*桶（Buckets）*

满足特定条件的文档的集合

*指标（Metrics）*

对桶内的文档进行统计计算

这就是全部了！每个聚合都是一个或者多个桶和零个或者多个指标的组合。翻译成粗略的SQL语句来解释吧：

```sql
SELECT COUNT(color) 
FROM table
GROUP BY color
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggs-high-level.html#CO183-1) | `COUNT(color)`  相当于指标。 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggs-high-level.html#CO183-2) | `GROUP BY color` 相当于桶。 |

桶在概念上类似于 SQL 的分组（GROUP BY），而指标则类似于 `COUNT()` 、 `SUM()` 、 `MAX()` 等统计方法。
```json
{
  "aggs": {
    "sales_over_time": {
      "date_histogram": {
        "field": "processDate",
        "interval": "year",
        "format": "yyyy"
      }
    }
  }
}
```