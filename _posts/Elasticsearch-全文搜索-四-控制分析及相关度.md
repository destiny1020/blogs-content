---
title: '[Elasticsearch] 全文搜索 (四) - 控制分析及相关度'
date: 2014-12-06 10:38:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 全文搜索]
---

## 控制分析(Controlling Analysis)

查询只能摘到真实存在于倒排索引(Inverted Index)中的词条(Term)，因此确保相同的分析过程会被适用于文档的索引阶段和搜索阶段的查询字符串是很重要的，这样才能够让查询中的词条能够和倒排索引中的词条匹配。

尽管我们说的是文档(Document)，解析器(Analyzer)是因字段而异的(Determined per Field)。每个字段都能够拥有一个不同的解析器，通过为该字段配置一个特定的解析器或者通过依赖类型(Type)，索引(Index)或者节点(Node)的默认解析器。在索引时，一个字段的值会被该字段的解析器解析。

<!-- More -->

比如，让我们为my_index添加一个新字段：

```json
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "properties": {
            "english_title": {
                "type":     "string",
                "analyzer": "english"
            }
        }
    }
}
```

现在我们就能够通过analyze API来比较english_title字段和title字段在索引时的解析方式，以Foxes这一单词为例：

```
GET /my_index/_analyze?field=my_type.title   
Foxes

GET /my_index/_analyze?field=my_type.english_title 
Foxes
```

对于title字段，它使用的是默认的standard解析器，它会返回词条foxes。 对于english_title字段，它使用的是english解析器，它会返回词条fox。

这说明当我们为词条fox执行一个低级的term查询时，english_title字段能匹配而title字段不能。

类似match查询的高阶查询能够理解字段映射(Field Mappings)，同时能够为查询的每个字段适用正确的解析器。我们可以通过validate-query API来验证这一点：

```json
GET /my_index/my_type/_validate/query?explain
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title":         "Foxes"}},
                { "match": { "english_title": "Foxes"}}
            ]
        }
    }
}
```

它会返回这个explanation：

> (title:foxes english_title:fox)

match查询会为每个字段适用正确的解析器，来确保该字段的查询词条的形式是正确的。

### 默认解析器(Default Analyzers)

尽管我们能够为字段指定一个解析器，但是当没有为字段指定解析器时，如何决定字段会使用哪个解析器呢？

解析器可以在几个级别被指定。ES会依次检查每个级别直到它找到了一个可用的解析器。在索引期间，检查的顺序是这样的：

- 定义在字段映射中的analyzer
- 文档的_analyzer字段中定义的解析器
- type默认的analyzer，它的默认值是
- 在索引设置(Index Settings)中名为default的解析器，它的默认值是
- 节点上名为default的解析器，它的默认值是
- standard解析器

在搜索期间，顺序稍微有所不同：

- 直接定义在查询中的analyzer
- 定义在字段映射中的analyzer
- type默认的analyzer，它的默认值是
- 在索引设置(Index Settings)中名为default的解析器，它的默认值是
- 节点上名为default的解析器，它的默认值是
- standard解析器

> NOTE
> 
> 以上两个斜体表示的项目突出显示了索引期间和搜索期间顺序的不同之处。_analyzer字段允许你能够为每份文档指定一个默认的解析器(比如，english，french，spanish)，而查询中的analyzer参数则让你能够指定查询字符串使用的解析器。然而，这并不是在一个索引中处理多语言的最佳方案，因为在[处理自然语言](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/languages.html)中提到的一些陷阱。

偶尔，在索引期间和查询期间使用不同的解析器是有意义的。比如，在索引期间我们也许希望能够索引同义词(Synonyms)(比如，对每个出现的quick，同时也索引fast，rapid和speedy)。但是在查询期间，我们必须要搜索以上所有的同义词。相反我们只需要查询用户输入的单一词汇，不管是quick，fast，rapid或者speedy。

为了实现这个区别，ES也支持index_analyzer和search_analyzer参数，以及名为default_index和default_search的解析器。

将这些额外的参数也考虑进来的话，索引期间查找解析器的完整顺序是这样的：

- 定义在字段映射中的index_analyzer
- 定义在字段映射中的analyzer
- 定义在文档_analyzer字段中的解析器
- type的默认index_analyzer，它的默认值是
- type的默认analyzer，它的默认值是
- 索引设置中default_index对应的解析器，它的默认值是
- 索引设置中default对应的解析器，它的默认值是
- 节点上default_index对应的解析器，它的默认值是
- 节点上default对应的解析器，它的默认值是
- standard解析器

而查询期间的完整顺序则是：

- 直接定义在查询中的analyzer
- 定义在字段映射中的search_analyzer
- 定义在字段映射中的analyzer
- type的默认search_analyzer，它的默认值是
- type的默认analyzer，它的默认值是
- 索引设置中的default_search对应的解析器，它的默认值是
- 索引设置中的default对应的解析器，它的默认值是
- 节点上default_search对应的解析器，它的默认值是
- 节点上default对应的解析器，它的默认值是
- standard解析器

### 配置解析器

能够指定解析器的地方太多也许会吓到你。但是实际上，它是非常简单的。

使用索引设置，而不是配置文件

第一件需要记住的是，即使你是因为一个单一的目的或者需要为一个例如日志的应用而使用ES的，很大可能你在将来会发现更多的用例，因此你会在相同的集群上运行多个独立的应用。每个索引都需要是独立的并且被独立地配置。

所以这就排除了在节点上配置解析器的必要。另外，在节点上配置解析器会改变每个节点的配置文件并且需要重启每个节点，这是维护上的噩梦。让ES持续运行并且只通过API来管理设置是更好的主意。

保持简单(Keep it simple)

多数时候，你都能提前知道你的文档会包含哪些字段。最简单的办法是在创建索引或者添加类型映射时为每个全文字段设置解析器。尽管该方法稍微有些繁琐，它能够让你清晰地看到每个字段上使用的解析器。

典型地，大多数字符串字段都会是精确值类型的not_analyzed字段，比如标签或者枚举，加上一些会使用standard或者english以及其他语言解析器的全文字段。然后你会有一两个字段需要自定义解析：比方title字段的索引方式需要支持"输入即时搜索(Find-as-you-type)"。

你可以在索引中设置default解析器，用来将它作为大多数全文字段的解析器，然后对某一两个字段配置需要的解析器。如果在你的建模中，你需要为每个类型使用一个不同的默认解析器，那么就在类型级别上使用analyzer设置。

> NOTE
> 
> 对于日志这类基于时间的数据，一个常用的工作流是每天即时地创建一个新的索引并将相应数据索引到其中。尽管这个工作流会让你无法预先创建索引，你仍然可以使用[索引模板(Index Templates)](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/indices-templates.html)来为一个新的索引指定其设置和映射。

## 相关度出问题了(Relevance is Broken)

在继续讨论更加复杂的[多字段查询](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/multi-field-search.html)前，让我们先快速地解释一下为何将[测试索引创建成只有一个主分片的索引](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/match-query.html#match-test-data)。

经常有新的用户反映称相关度出问题了，并且提供了一个简短的重现：用户索引了一些文档，运行了一个简单的查询，然后发现相关度低的结果明显地出现在了相关度高的结果的前面。

为了弄明白它发生的原因，让我们假设我们创建了一个拥有两个主分片的索引，并且索引了10份文档，其中的6份含有单词foo。在分片1上可能保存了包含单词foo的3份文档，在分片2上保存了另外3份。换言之，文档被均匀的分布在分片上。

在[什么是相关度](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/relevance-intro.html)一节中，我们描述了在ES中默认使用的相似度算法 - 即词条频度/倒排文档频度(Term Frequency/Inverse Document Frequency，TF/IDF)。词条频度计算词条在当前文档的对应字段中出现的次数。词条出现的次数越多，那么该文档的相关度就越高。倒排文档频度将一个词条在索引的所有文档中出现程度以百分比的方式考虑在内。词条出现的越频繁，那么它的权重就越小。

但是，因为性能上的原因，ES不会计算索引中所有文档的IDF。相反，每个分片会为其中的文档计算一个本地的IDF。

因为我们的文档被均匀地分布了，两个分片上计算得到的IDF应该是相同的。现在想象一下如果含有foo的5份文档被保存在了分片1上，而只有1份含有foo的文档被保存在了分片2上。在这种情况下，词条foo在分片1上就是一个非常常见的词条(重要性很低)，但是在分片2上，它是非常少见的词条(重要性很高)。因此，这些IDF的差异就会导致错误的结果。

实际情况下，这并不是一个问题。当你向索引中添加的文档越多，本地IDF和全局IDF之间的差异就会逐渐减小。考虑到真实的世界中的数据量，本地IDF很快就会变的正常。问题不是相关度，而是数据量太小了。

对于测试，有两种方法可以规避该问题。第一种方法是创建只有一个主分片的索引，正如我们在介绍[match查询](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/match-query.html)时做的那样。如果你只有一个分片，那么本地IDF就是全局IDF。

第二种方法是在搜索请求中添加?search_type=dfs_query_then_fetch。dfs表示分布频度搜索(Distributed Frequency Search)，它会告诉ES首先从每个分片中获取本地IDF，然后计算整个索引上的全局IDF。

> TIP
> 
> 不要在生产环境中使用dfs_query_then_fetch。它真的是不必要的。拥有足够的数据就能够确保你的词条频度会被均匀地分布。没有必要再为每个查询添加这个额外的DFS步骤。
