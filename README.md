# Introduction

书籍

[https://es.xiaoleilu.com/075\_Inside\_a\_shard/60\_Segment\_merging.html](https://es.xiaoleilu.com/075_Inside_a_shard/60_Segment_merging.html)



**一、ElasticSearch使用场景**  
**存储**  
ElasticSearch天然支持分布式，具备存储海量数据的能力，其搜索和数据分析的功能都建立在ElasticSearch存储的海量的数据之上；ElasticSearch很方便的作为海量数据的存储工具，特别是在数据量急剧增长的当下，ElasticSearch结合爬虫等数据收集工具可以发挥很大用处

**搜索**  
ElasticSearch使用倒排索引，每个字段都被索引且可用于搜索，更是提供了丰富的搜索api，在海量数据下近实时实现近秒级的响应,基于Lucene的开源搜索引擎，为搜索引擎（全文检索，高亮，搜索推荐等）提供了检索的能力。 具体场景:  
**1.**Stack Overflow（国外的程序异常讨论论坛），IT问题，程序的报错，提交上去，有人会跟你讨论和回答，全文检索，搜索相关问题和答案，程序报错了，就会将报错信息粘贴到里面去，搜索有没有对应的答案；  
**2.**GitHub（开源代码管理），搜索上千亿行代码；  
**3.**电商网站，检索商品；  
**4.**日志数据分析，logstash采集日志，ElasticSearch进行复杂的数据分析（ELK技术，elasticsearch+logstash+kibana）；

**数据分析**  
ElasticSearch也提供了大量数据分析的api和丰富的聚合能力，支持在海量数据的基础上进行数据的分析和处理。具体场景：  
爬虫爬取不同电商平台的某个商品的数据，通过ElasticSearch进行数据分析（各个平台的历史价格、购买力等等）；

**二、ElasticSearch架构**

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724100748331-1344625827.png)

**1.**Gateway是ES用来存储索引的文件系统，支持多种类型。  
**2.**Gateway的上层是一个分布式的lucene框架。  
**3.**Lucene之上是ES的模块，包括：索引模块、搜索模块、映射解析模块等  
**4.**ES模块之上是 Discovery、Scripting和第三方插件。Discovery是ES的节点发现模块，不同机器上的ES节点要组成集群需要进行消息通信，集群内部需要选举master节点，这些工作都是由Discovery模块完成。支持多种发现机制，如 Zen 、EC2、gce、Azure。Scripting用来支持在查询语句中插入javascript、python等脚本语言，scripting模块负责解析这些脚本，使用脚本语句性能稍低。ES也支持多种第三方插件。  
**5.**再上层是ES的传输模块和JMX.传输模块支持多种传输协议，如 Thrift、memecached、http，默认使用http。JMX是java的管理框架，用来管理ES应用。  
**6.**最上层是ES提供给用户的接口，可以通过RESTful接口和ES集群进行交互。

\[\[ 小提示：目前市场上开放源代码的最好全文检索引擎工具包就属于 Apache 的 Lucene了。但是 Lucene 只是一个工具包，它不是一个完整的全文检索引擎。Lucene 的目的是为软件开发人员提供一个简单易用的工具包，以方便的在目标系统中实现全文检索的功能，或者是以此为基础建立起完整的全文检索引擎。目前以 Lucene 为基础建立的开源可用全文搜索引擎主要是 Solr 和 Elasticsearch。Solr 和 Elasticsearch 都是比较成熟的全文搜索引擎，能完成的功能和性能也基本一样。但是 ES 本身就具有分布式的特性和易安装使用的特点，而 Solr 的分布式需要借助第三方来实现，例如通过使用 ZooKeeper 来达到分布式协调管理。不管是 Solr 还是 Elasticsearch 底层都是依赖于 Lucene，而 Lucene 能实现全文搜索主要是因为它实现了倒排索引的查询结构。\]\]

