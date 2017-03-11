---
title: '[Elasticsearch] 多字段搜索 (六) - 自定义_all字段，跨域查询及精确值字段'
date: 2014-12-12 10:00:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 全文搜索]
---

## 自定义_all字段

在[元数据：_all字段](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/root-object.html#all-field)中，我们解释了特殊的_all字段会将其它所有字段中的值作为一个大字符串进行索引。尽管将所有字段的值作为一个字段进行索引并不是非常灵活。如果有一个自定义的_all字段用来索引人名，另外一个自定义的_all字段用来索引地址就更好了。

<!-- More -->

ES通过字段映射中的copy_to参数向我们提供了这一功能：

```json
PUT /my_index
{
    "mappings": {
        "person": {
            "properties": {
                "first_name": {
                    "type":     "string",
                    "copy_to":  "full_name" 
                },
                "last_name": {
                    "type":     "string",
                    "copy_to":  "full_name" 
                },
                "full_name": {
                    "type":     "string"
                }
            }
        }
    }
}
```

现在first_name和last_name字段中的值会被拷贝到full_name字段中。

有了这个映射，我们可以通过first_name字段查询名字，last_name字段查询姓氏，或者full_name字段查询姓氏和名字。

> NOTE
> 
> first_name和last_name字段的映射和full_name字段的索引方式的无关。full_name字段会从其它两个字段中拷贝字符串的值，然后仅根据full_name字段自身的映射进行索引。

## 跨域查询(Cross-fields Queries)

如果你在索引文档前就能够自定义_all字段的话，那么使用_all字段就是一个不错的方法。但是，ES同时也提供了一个搜索期间的解决方案：使用类型为cross_fields的multi_match查询。cross_fields类型采用了一种以词条为中心(Term-centric)的方法，这种方法和best_fields及most_fields采用的以字段为中心(Field-centric)的方法有很大的区别。它将所有的字段视为一个大的字段，然后在任一字段中搜索每个词条。

为了阐述以字段为中心和以词条为中心的查询的区别，看看以字段为中心的most_fields查询的解释(译注：通过validate-query API得到)：

```json
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "most_fields",
            "operator":    "and", 
            "fields":      [ "first_name", "last_name" ]
        }
    }
}
```

operator设为了and，表示所有的词条都需要出现。

对于一份匹配的文档，peter和smith两个词条都需要出现在相同的字段中，要么是first_name字段，要么是last_name字段：

> (+first_name:peter +first_name:smith) (+last_name:peter +last_name:smith)

而已词条为中心的方法则使用了下面这种逻辑：

> +(first_name:peter last_name:peter) +(first_name:smith last_name:smith)

换言之，词条peter必须出现在任一字段中，同时词条smith也必须出现在任一字段中。

cross_fields类型首先会解析查询字符串来得到一个词条列表，然后在任一字段中搜索每个词条。仅这个区别就能够解决在[以字段为中心的查询](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/field-centric.html)中提到的3个问题中的2个，只剩下倒排文档频度的不同这一问题。

幸运的是，cross_fields类型也解决了这个问题，从下面的validate-query请求中可以看到：

```json
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields", 
            "operator":    "and",
            "fields":      [ "first_name", "last_name" ]
        }
    }
}
```

它通过混合(Blending)字段的倒排文档频度来解决词条频度的问题：

> +blended("peter", fields: [first_name, last_name]) +blended("smith", fields: [first_name, last_name])

换言之，它会查找词条smith在first_name和last_name字段中的IDF值，然后使用两者中较小的作为两个字段最终的IDF值。因为smith是一个常见的姓氏，意味着它也会被当做一个常见的名字。

> NOTE
> 
> 为了让cross_fields查询类型能以最佳的方式工作，所有的字段都需要使用相同的解析器。使用了相同的解析器的字段会被组合在一起形成混合字段(Blended Fields)。
> 
> 如果你包含了使用不同解析链(Analysis Chain)的字段，它们会以和best_fields相同的方被添加到查询中。比如，如果我们将title字段添加到之前的查询中(假设它使用了一个不同的解析器)，得到的解释如下所示：
> 
> (+title:peter +title:smith) ( +blended("peter", fields: [first_name, last_name]) +blended("smith", fields: [first_name, last_name]) )
> 
> 当使用了minimum_should_match以及operator参数时，这一点尤为重要。

### 逐字段提升(Per-field Boosting)

使用cross_fields查询相比使用[自定义_all字段](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/custom-all.html)的一个优点是你能够在查询期间对个别字段进行提升。

对于first_name和last_name这类拥有近似值的字段，也许提升是不必要的，但是如果你通过title和description字段来搜索书籍，那么你或许会给予title字段更多的权重。这可以通过前面介绍的caret(^)语法来完成：

```json
GET /books/_search
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields",
            "fields":      [ "title^2", "description" ] 
        }
    }
}
```

能够对个别字段进行提升带来的优势应该和对多个字段执行查询伴随的代价进行权衡，因为如果使用自定义的_all字段，那么只需要要对一个字段进行查询。选择能够给你带来最大收益的方案。

## 精确值字段(Exact-value Fields)

在结束对于多字段查询的讨论之前的最后一个话题是作为not_analyzed类型的精确值字段。在multi_match查询中将not_analyzed字段混合到analyzed字段中是没有益处的。

原因可以通过validate-query进行简单地验证，假设我们将title字段设置为not_analyzed：

```json
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields",
            "fields":      [ "title", "first_name", "last_name" ]
        }
    }
}
```

因为title字段时没有被解析的，它会以将整个查询字符串作为一个词条进行搜索！

> title:peter smith ( blended("peter", fields: [first_name, last_name]) blended("smith", fields: [first_name, last_name]) )

很显然该词条在title字段的倒排索引中并不存在，因此永远不可能被找到。在multi_match查询中避免使用not_analyzed字段。
