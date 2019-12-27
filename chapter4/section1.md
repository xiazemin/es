# section1

# 1. 技术架构 {#1}

有赞搜索引擎基于分布式实时引擎elasticsearch\(ES\). ES构建在开源社区最稳定成熟的索引库lucence上, 支持多用户租用, 高可用, 可水平扩展; 并有自动容错和自动伸缩的机制. 我们同事还实现了es与mysql和hadoop的无缝集成; 我们自主开发了高级搜索模块提供灵活的相关性计算框架等功能.![](http://images2015.cnblogs.com/blog/759343/201603/759343-20160321183146464-1982557830.png "pic")

# 2. 索引构建 {#2}

互联网索引的特点是实时性高, 数据量大. 时效性要求用户和客户的各种行为能够第一时间进入索引; 数据量大要求一个有效分布式方案可以在常数时间内创建不断增长的TB数量级索引.

实时索引我们采用面向队列的架构, 数据首先写入DB\(或文件\), 然后通过数据库同步机制将数据流写入kafka队列. 这种同步机制和数据库主从同步的原理相同, 主要的开源产品有mypipe和阿里推出的canal. es通过订阅相应的topic实现实时建立索引.

如果数据源是文件, 则使用flume实时写入Kafka.

另外一个索引问题是全量索引. 有如下几个场景让全量索引是一个必要过程: 1. 实时更新有可能会丢数据, 每次很少的丢失时间长了降低搜索引擎的质量. 周期性的全量更新是解决这个问题的最直接的方法;  
2. 即使能够保证实时更新, 业务的发展有可能有重新建索引的需求\(比如增加字段, 修改属性, 修改分词算法等\).  
3. 很多搜索引擎是在业务开始后很久才搭建的, 冷启动必须全量创建索引.

我们采用Hadoop-es利用hadoop分布式的特性来创建索引. hadoop-es让分布式索引对用户透明, 就像单机更新索引一样. 一个是分布式的数据平台, 一个是分布式搜索引擎, 如果能把这两个结合就能够实现分布式的全量索引过程. Hadoop-es正式我们想要的工具.

![](http://images2015.cnblogs.com/blog/759343/201603/759343-20160321183147261-459829963.jpg)

我们给出一个通过Hive sql创建索引的例子:

drop table search.goods\_index;  

CREATE EXTERNAL TABLE search.goods\_index \(  

    is\_virtual int,

    created\_time string,

    update\_time string,

    title string,

    tag\_ids array&lt;int&gt;

  \) STORED BY 'org.elasticsearch.hadoop.hive.EsStorageHandler' TBLPROPERTIES \(

    'es.batch.size.bytes'='1mb',

    'es.batch.size.entries'='0',

    'es.batch.write.refresh'='false',

    'es.batch.write.retry.count'='3',

    'es.mapping.id'='id',

    'es.write.operation'='index',

    'es.nodes'='192.168.1.10:9200',

    'es.resource'='goods/goods'\);

系统把es映射成hive的一个外部表, 更新索引就像是写入一个hive表一样. 实际上所有分布式问题都被系统透明了.



不建议从数据库或文件系统来全量索引. 一方面这会对业务系统造成很大的压力, 另一方面因为数据库和文件系统都不是真正分布式系统, 自己写程序保证全量索引的水平扩展性很容易出问题, 也没有必要这么做.



全量索引和增量索引的架构如下图所示. 另外一点是hadoop也是订阅kafka备份数据库和日志的. 我个人建议一个公司所有DB和文件都存储在hadoop上, 这样做起码有2个好处: 1. hadoop上使用hive或者spark创建的数据仓库为大数据提供统一的操作接口.

2. hadoop数据相对于线上更加稳定, 可以作为数据恢复的最后一个防线.

数据仓库的话题不在本篇文章的讨论范围, 这里只是简单提一下.

