---
title: '[Elasticsearch] 多字段搜索 (二) - 最佳字段查询及其调优'
date: 2014-12-09 10:23:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 全文搜索]
---

## 最佳字段(Best Fields)

假设我们有一个让用户搜索博客文章的网站，就像这两份文档一样：

```json
PUT /my_index/my_type/1
{
    "title": "Quick brown rabbits",
    "body":  "Brown rabbits are commonly seen."
}

PUT /my_index/my_type/2
{
    "title": "Keeping pets healthy",
    "body":  "My quick brown fox eats rabbits on a regular basis."
}
```

<!-- More -->

用户输入了"Brown fox"，然后按下了搜索键。我们无法预先知道用户搜索的词条会出现在博文的title或者body字段中，但是用户是在搜索和他输入的单词相关的内容。以上的两份文档中，文档2似乎匹配的更好一些，因为它包含了用户寻找的两个单词。

让我们运行下面的bool查询：

```json
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```

然后我们发现文档1的分值更高：

```json
{
  "hits": [
     {
        "_id":      "1",
        "_score":   0.14809652,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     },
     {
        "_id":      "2",
        "_score":   0.09256032,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     }
  ]
}
```

要理解原因，想想bool查询是如何计算得到其分值的：

1. 运行should子句中的两个查询
2. 相加查询返回的分值
3. 将相加得到的分值乘以匹配的查询子句的数量
4. 除以总的查询子句的数量

文档1在两个字段中都包含了brown，因此两个match查询都匹配成功并拥有了一个分值。文档2在body字段中包含了brown以及fox，但是在title字段中没有出现任何搜索的单词。因此对body字段查询得到的高分加上对title字段查询得到的零分，然后在乘以匹配的查询子句数量1，最后除以总的查询子句数量2，导致整体分值比文档1的低。

在这个例子中，title和body字段是互相竞争的。我们想要找到一个最佳匹配(Best-matching)的字段。

如果我们不是合并来自每个字段的分值，而是使用最佳匹配字段的分值作为整个查询的整体分值呢？这就会让包含有我们寻找的两个单词的字段有更高的权重，而不是在不同的字段中重复出现的相同单词。

### dis_max查询

相比使用bool查询，我们可以使用dis_max查询(Disjuction Max Query)。Disjuction的意思"OR"(而Conjunction的意思是"AND")，因此Disjuction Max Query的意思就是返回匹配了任何查询的文档，并且分值是产生了最佳匹配的查询所对应的分值：

```json
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```

它会产生我们期望的结果：

```json
{
  "hits": [
     {
        "_id":      "2",
        "_score":   0.21509302,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id":      "1",
        "_score":   0.12713557,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     }
  ]
}
```

## 最佳字段查询的调优

如果用户搜索的是"quick pets"，那么会发生什么呢？两份文档都包含了单词quick，但是只有文档2包含了单词pets。两份文档都没能在一个字段中同时包含搜索的两个单词。

一个像下面那样的简单dis_max查询会选择出拥有最佳匹配字段的查询子句，而忽略其他的查询子句：

```json
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ]
        }
    }
}
```

```json
{
  "hits": [
     {
        "_id": "1",
        "_score": 0.12713557, 
        "_source": {
           "title": "Quick brown rabbits",
           "body": "Brown rabbits are commonly seen."
        }
     },
     {
        "_id": "2",
        "_score": 0.12713557, 
        "_source": {
           "title": "Keeping pets healthy",
           "body": "My quick brown fox eats rabbits on a regular basis."
        }
     }
   ]
}
```
可以发现，两份文档的分值是一模一样的。

我们期望的是同时匹配了title字段和body字段的文档能够拥有更高的排名，但是结果并非如此。需要记住：dis_max查询只是简单的使用最佳匹配查询子句得到的_score。

### tie_breaker

但是，将其它匹配的查询子句考虑进来也是可能的。通过指定tie_breaker参数：

```json
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ],
            "tie_breaker": 0.3
        }
    }
}
```

它会返回以下结果：

```json
{
  "hits": [
     {
        "_id": "2",
        "_score": 0.14757764, 
        "_source": {
           "title": "Keeping pets healthy",
           "body": "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id": "1",
        "_score": 0.124275915, 
        "_source": {
           "title": "Quick brown rabbits",
           "body": "Brown rabbits are commonly seen."
        }
     }
   ]
}
```

现在文档2的分值比文档1稍高一些。

tie_breaker参数会让dis_max查询的行为更像是dis_max和bool的一种折中。它会通过下面的方式改变分值计算过程：

1. 取得最佳匹配查询子句的_score。
2. 将其它每个匹配的子句的分值乘以tie_breaker。
3. 将以上得到的分值进行累加并规范化。

通过tie_breaker参数，所有匹配的子句都会起作用，只不过最佳匹配子句的作用更大。

> NOTE
> 
> tie_breaker的取值范围是0到1之间的浮点数，取0时即为仅使用最佳匹配子句(译注：和不使用tie_breaker参数的dis_max查询效果相同)，取1则会将所有匹配的子句一视同仁。它的确切值需要根据你的数据和查询进行调整，但是一个合理的值会靠近0，(比如，0.1 -0.4)，来确保不会压倒dis_max查询具有的最佳匹配性质。
