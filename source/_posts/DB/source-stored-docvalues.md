---
title: ElasticSearch中_source、store_fields、doc_values性能比较
date: 2021-10-22
categories: 数据库
mathjax: false
tags: 
- DB
- ElasticSearch
- Lucene
keywords:
- _source
- store_fields
- doc_values
- ElasticSearch性能测试
---

在这篇文章中，我想**从性能的角度探讨elasticsearch 为我们存储了哪些字段，以及在查询检索时提供了哪些可用的字段**。实际上，ElasticSearch和Solr的底层库Lucene提供了两种存储和检索字段的方式：stored和doc_values。此外，ElasticSearch默认提供了 `_source` 字段，这是在索引时由文档的所有字段构造的一个大json。

为什么 ElasticSearch使用 `_source` 字段作为默认值，所有这些可用的字段从性能的角度来看有什么区别？让我们一探究竟！

# lucene中的stored和doc_values

当我们在 Lucene 中索引一个文档时，已经被索引的原始字段的信息丢失了。字段根据schema配置进行分词、转换然后索引形成倒排索引。没有任何额外的数据结构，当我们搜索一个文档时，我们得到的是被搜索文档的 id 而不是原始字段。为了获得这些原始信息，我们需要额外的数据结构。Lucene为此提供了两种可用的方式：Stored Field和doc_values。

## Stored Field

Stored Field的目的是存储字段的原始值（没被分词），以便在查询时检索它们。

## doc_values

引入了 doc_values 是为了对一些操作进行加速，例如faceting、sorting和grouping。doc_values 也可用于在查询时返回字段值。唯一的限制是我们不能在text字段上使用doc_values。

Stored Field和doc_values是在 Lucene 库中实现的，它们可以在 Solr 和 ElasticSearch 中使用。

我写了一篇博文，比较了 Solr 中Stored Field和doc_values检索性能：

[DocValues VS Stored Fields : Apache Solr Features and Performance SmackDown](https://sease.io/2020/03/docvalues-vs-stored-fields-apache-solr-features-and-performance-smackdown.html).

您可以找到关于存储字段和文档值、它们的使用和约束的更详细的描述。

# ElasticSearch中的字段检索

如果我们在映射中明确定义存储的字段和文档值，则可以在 elasticsearch 中使用它们：

```json
"properties" : {
    "field": {
        "type": "keyword",
        "store": true,
        "doc_values" true
    }
}
```

默认情况下，[每个字段的`store`都设置为 false](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-store.html)。相反，[所有支持`doc_values`的字段都会默认开启`doc_values`](https://www.elastic.co/guide/en/elasticsearch/reference/current/doc-values.html)。

根据Stored Field和doc_values的默认配置，在查询时仍然会返回查询命中的文档中的每个字段值。发生这种情况是因为 ElasticSearch 使用另一种工具进行字段检索：**elasticsearch _source**字段。

## elasticsearch _source 字段

`_source` 字段是在索引时传递给 ElasticSearch 的 json。此字段在 ElasticSearch 中默认设置为 true，可以通过以下方式使用`mappings`禁用：

```json
"mappings": {
    "_source": {
        "enabled": false
    }
}
```

查询时默认返回所有字段。您也可以[指定要在响应中返回的`_source`中的一部分字段](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/search-fields.html#source-filtering)，这应该是为了提高网络传输的响应速度。

```http
GET /_search
{
  "_source": {
    "includes": [ "obj1.*", "obj2.*" ],
    "excludes": [ "*.description" ]
  },
  "query": {
    "term": {
      "user.id": "kimchy"
    }
  }
}
```

可以[通过适当的配置将`_source`的某些字段排除](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html#include-exclude)：

```http
PUT logs
{
  "mappings": {
    "_source": {
      "excludes": [
        "meta.description",
        "meta.other.*"
      ]
    }
  }
}
```

从`_source`中排除字段将减少磁盘空间使用，但被排除的字段将永远不会在响应中返回。

禁用 elasticsearch `_source` 字段，更新文档需要从头开始重新索引。实际上，为了更新文档，我们需要从旧文档中获取字段的值。从逻辑上讲，使用Stored Field和doc_values从旧文档中获取字段的值应该是可行的（这就是 Solr 中原子更新的工作方式）。但是，由于设计决定，这在 ElasticSearch 中是不允许的，如果您需要更新文档，则必须在 elasticsearch 索引配置中启用`_source`字段。

## 检索字段

在 elasticsearch 中，您可以启用或禁用`_source`字段并使Stored Field或`doc_values`。但是如何在查询时检索字段？

默认情况下，如果启用了`_source`，则返回包含整个文档的`_source`。您可以避免它并仅返回源的一个子集，如下所示：

```json
POST my-index-000001/_search
{
  "query": {
    "match": {
      "user.id": "kimchy"
    }
  },
  "fields": [
    "user.id",
    "http.response.*",         
    {
      "field": "@timestamp",
      "format": "epoch_millis" 
    }
  ],
  "_source": false
}
```

但是，如果您没有启用`_source`字段，并且想要从Stored Field和`doc_values`返回字段，则必须以另一种方式告诉它给 ElasticSearch。对于您使用的每个源，您必须以不同的方式指定字段列表：

```json
...
 "fields": ["sv1", "sv2",...],
 "docvalue_fields": ["dv1", "dv2",...],
 "stored_fields" : ["s1", "s2",...],
...
```

例如，如果您有一个字段既存储了Stored Field也存储了doc_values，您可以选择是从Stored Field还是doc_values中检索它。从功能的角度来看，这完全相同，但您的选择可能会影响查询的执行时间。

# stored 字段, doc_values和ElasticSearch _source内部形式

在本节中，我只想简要概述Stored Field、`_source` 字段和 `doc_values` 的内部结构，以便获得一些工具来了解使用这些方法进行字段检索时对性能的期望。

## stored内部结构

存储的字段以行方式放置在磁盘中：对于每个文档，我们有一行连续包含所有存储的字段。

![img](https://i0.wp.com/sease.io/wp-content/uploads/2020/11/storedfields.png?resize=716%2C160&ssl=1)

以上图为例。为了访问文档 x 的 field3，我们必须访问文档 x 的行并跳过存储在 field3 之前的所有字段。跳过字段需要获取其长度。跳过字段虽然不像读取那么繁琐，但此操作并非不耗时。

## Doc_values内部结构

文档值以列方式存储。不同文档的相同字段的值一起连续存储在内存中，并且可以“几乎”直接访问某个文档的某个字段。计算一个想要的值的地址不是一个简单的操作，它有一个计算成本，但我们可以想象，如果我们只想要一个字段，使用这种访问会更有效率。

## ElasticSearch _source内部结构

`_source` 呢？好吧，如上所述，源是一个包含 json 的大字段，其中包含在索引时提供给 ElasticSearch 的所有输入。但是，这个字段实际上是如何存储的？毫不奇怪，elasticsearch 利用了一种已经由 lucene 实现和提供的机制：Stored Field。而且，`_source` 字段是行中第一个存储的字段。

![img](https://i0.wp.com/sease.io/wp-content/uploads/2020/11/sourcefield.jpg?resize=707%2C175&ssl=1)

必须阅读整个 _source 才能使用它包含的信息。如果我们要返回一个文档的所有字段，这个过程直观上是最快的。另一方面，如果我们只需要返回它包含的信息的一小部分，读取这个巨大的字段可能会浪费计算能力。

# 性能测试

为了对 3 种类型的字段进行基准测试，我在 ElasticSearch 中创建了 3 个不同的索引。我索引了来自维基百科的 100 万个文档，对于每个文档，我用三种不同的方法索引了 100 个包含 15 个字符的字符串字段：在第一个索引中，我将字段设置为存储，在第二个索引中设置为 docvalues。在这两个索引中，我都禁用了源字段。相反，在第三个索引中，我只是启用了源字段。

文档和查询集合来自 https://github.com/tantivy-search/search-benchmark-game。 我使用真实的集合来模拟真实的场景。 

执行细节：

- CPU: AMD锐龙3600
- RAM: 32 GB

对于每个查询，我请求了最好的 200 个文档，并重复测试，将要返回的字段数量（在我创建的 100 个随机字符串字段中）从 1 更改为 100。

这是基准测试的结果：

![img](https://i0.wp.com/sease.io/wp-content/uploads/2020/11/elastic-benachmarks.png?resize=706%2C530&ssl=1)

结果正好显示了我们期望看到的结果。**如果我们需要每个文档的字段很少，建议使用 Docvalues**。另一方面，**当我们想要返回整个文档_source 字段是最好的**，而`store_field`的使用是其他两者之间的完美折衷。

在我执行的基准测试场景中，**如果我们只需要一个字段**，**docvalues 的速度几乎是 _source 字段的两倍** ，而在相反的极端情况下，如果我们想返回所有字段，图表显示当我们需要时，速度几乎提高了 **2倍**使用 _source 字段代替 docvalues。

总之，性能不是我们必须考虑的唯一参数。正如我们在这篇博文中简要解释的那样，使用一种或另一种方法存在一些限制。由于您的用例的一些限制，您可能被迫使用这三个中的一个。而且即使从表现来看，我们也没有明显的赢家。

如果磁盘空间不是问题，**您甚至可以混合不同的方法并将字段设置为存储和文档值，并保持_source启用**。在查询时，elasticsearch 使您可以选择所需的字段列表，以及是否希望从 `_source`、`_store_field`或 `doc_values` 返回它们。





参考：

https://www.elastic.co/guide/en/elasticsearch/reference/current/search-fields.html

https://sease.io/2021/02/field-retrieval-performance-in-elasticsearch.html