这半年业余时间大部分都在研究Lucene的索引结构，苦于寻觅这方面的相关书籍，唯一一本较好的《[Lucene In Action](https://book.douban.com/subject/6440615/)》最新版也是Lucene3.0为主，然而官网已经更新到[Lucene7.6](http://lucene.apache.org/core/7_6_0/changes/Changes.html)了。只能看看源码与相关博客来学习，这里记录下我对Lucene的一些认识。

# 1、 前缀树



![Trie](http://ww1.sinaimg.cn/large/bda5cd74ly1fzffqu4719j20c80ogtc1.jpg)

前缀树的一个应用场景是自动补全，这是很多搜索引擎的必备功能。

![自动补全](http://ww1.sinaimg.cn/large/bda5cd74ly1fzffw3xvc0g20dw06o77x.gif)

前缀树的缺点：

1、空间使用率不高

# 2、 压缩前缀树

![Compact Trie](http://ww1.sinaimg.cn/large/bda5cd74ly1fzffsmmr29j20m60m0n0d.jpg)



# 3、FST

![FST](http://ww1.sinaimg.cn/large/bda5cd74ly1fzdd8e6up8j20t9084dgr.jpg)







# 4、 FST在Lucene中的应用

@startuml

class AutomatonQuery
AutomatonQuery <|-- RegexpQuery
AutomatonQuery <|-- TermRangeQuery
AutomatonQuery <|-- PrefixQuery
AutomatonQuery <|-- WildcardQuery

@enduml





http://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html

http://blog.mikemccandless.com/2013/06/build-your-own-finite-state-transducer.html

https://blog.burntsushi.net/transducers/

https://stackoverflow.com/questions/2602253/how-does-lucene-index-documents/43203339

http://blog.51cto.com/sbp810050504/1361551

https://en.wikipedia.org/wiki/Trie

http://examples.mikemccandless.com/fst.py

https://issart.com/blog/full-text-search-how-it-works/
