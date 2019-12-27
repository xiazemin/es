# ELK在广告系统监控中的应用

[**E**lasticsearch](https://www.elastic.co/products/elasticsearch)

* : 分布式，实时，全文搜索引擎，最核心的部分，也是接下来主要介绍的内容
* [**L**ogstash](https://www.elastic.co/products/logstash)
  : 非常灵活的日志收集工具，不局限于向 Elasticsearch 导入数据，可以定制多种输入，输出，及过滤转换规则
* [**K**ibana](https://www.elastic.co/products/kibana)
  : 提供对于 Elasticsearch 数据的搜索及可视化功能。并且支持开发人员自己按需开发插件。

# ELK在广告系统监控中的应用 {#elk在广告系统监控中的应用}

广告系统对于请求的响应时间非常敏感，此外，对于并发请求数要求也很高。因此，我们需要从请求响应时间，以及QPS两个指标，来检测系统的性能; 同时，为了定位瓶颈，我们需要把这两个指标，分拆到请求处理过程中的每个组件去看。

此外，为了优化全球用户访问的速度，我们在全球部署了多个节点。因此，整个系统不是在一个内网环境，对监控数据的实时收集提出了考验。

因此，我们需要一套灵活的监控统计库，能够非侵入的注册到业务代码中; 并且在一个统一的入口，实时监控每个节点的运行情况。

![](https://cloud.githubusercontent.com/assets/839287/13384488/f2339846-ded0-11e5-8b62-368e9d4de3de.png "elk")

我们采用方案是:

* 每个节点，有一个 Collector，负责收集监控数据，按照节点/服务/时间维度做聚合，实时写入 Kinesis 队列
* Logstash 将 Kinesis 中的监控数据，实时导入到 Elasticsearch
* Kibana 后台上，通过定制查询图表，可以统计对比每个节点/服务/组件的监控数据
* 我们采用默认的 Logstash 日志导入策略，按天索引，使用
  [curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html)
  工具，定期对历史数据删除/优化。
* 此外，通过
  [Watcher](https://www.elastic.co/products/watcher)
  插件，定制告警信息，从而在系统出现问题时及时处理。

这套机制，很好地满足了我们对于系统监控的需求，帮助我们分析，定位优化系统瓶颈。

Elasticsearch简介

Elasticsearch 是一个分布式，实时，全文搜索引擎。所有操作都是通过 RESTful 接口实现; 其底层实现是基于 Lucene 全文搜索引擎。数据以JSON文档的格式存储索引，不需要预先规定范式。



和传统数据库的术语对比一下，也许能够帮助我们对 Elasticsearch 有一个更加感性的认识



RDS	Elasticsearch	说明

database	index	

table	type	

primary key	id	

row	JSON document	文档是最基本的数据存储单元，因此也可以称 Elasticsearch 为文档型 NoSQL 数据库

column	field	

schema	mapping	可以支持动态范式，也可以将范式指定下来，从而优化对数据索引和查询

index	\(all\)	所有的字段都是被索引的，因此不需要手动指定索引

SQL	query DSL	不同于我们习惯的SQL语句，需要构造繁冗的JSON参数来实现复杂的数据统计查询功能

使用简介

所有操作都是通过RESTfull接口完成。请求URI指定了文档的"路径"。 注意到语义通过请求的HTTP方法不同区分开来:



创建 POST /{index}/{type} {"field": "value", ...}

创建/更新 PUT /{index}/{type}/{id} {"field": "value", ...}

判断一个文档是否存在 HEAD /{index}/{type}/{id}

获取一个文档 GET /{index}/{type}/{id}

删除 DELETE /{index}/{type}/{id}

说明一下POST和PUT的区别: POST 永远是创建新的。PUT 可以表示创建，但是如果指定的URI存在，则含义为更新。换句话说，一个PUT请求，重复执行，结果应该是一样的。因此，在 Elasticsearch 的API，POST 表示创建新的文档，并由 Elasticsearch 自动生成id; 而 PUT 方法需要指定文档id，含义为"创建，若存在则更新"。



另外，需要提一下版本的概念。因为 Elasticsearch 中的文档是不可变的\(immutable\)，所以每个文档会有一个 版本号\(version\)字段，每次更新，实际上是将旧的版本标记未删除，并创建了一个新版本\(版本号+1\)。这个开销是很大的。因此，在实际使用中，需要尽量避免频繁的更新操作。



为了满足一些复杂的数据统计，仅仅上述的增删改查是不够的，为了充分使用 Elasticsearch的搜索功能，还需要学习 query DSL 的使用。query DSL 写起来比较复杂，这里仅仅列举一下和SQL关键字的对应。具体使用还得参考文档。



SQL            \| query DSL

-------        \| --------

=              \| {"term": {field: val}

IN             \| {"terms": {field: \[val, ...\]}

LIKE           \| {"wildcard:" {field: pattern}}

BETWEEN AND    \| {"range": {field: {"gt": val, "lt": val}}}

AND / OR / NOT \| {"bool": {"must"/"should"/"must\_not": ...}

Aggregations   \| {"aggs": ...}

JOIN           \| {"nestted"/"has\_child"/"has\_parent": ...}

随便列两个查询语句，感受一下:



SELECT \* FROM megacorp.employee WHERE age &gt; 30 AND last\_name = "smith"



GET /megacorp/employee/\_search

{

  "query": {

    "filtered": {

      "filter": {

        "range": { "age": { "gt": 30 } }

      },

      "query": {

        "match": {

          "last\_name": "smith"

        }

      }

    }

  }

}

SELECT interests，avg\(age\) FROM megacorp.employee GROUP BY interests



GET /megacorp/employee/\_search

{

  "aggs": {

    "all\_interests": {

      "terms": { "field": "interests" },

      "aggs": {

        "avg\_age": {

          "avg": {

            "field": "age"

          }

        }

      }

    }

  }

}

从个人的使用经验来说，熟练掌握 query DSL 的难度还是不小的 \(毕竟大家都习惯的SQL的简洁直接\)。此外，由于输入输出都是嵌套很深的JSON，解析起来也比较麻烦。为了降低使用门槛，一般都会有从SQL翻译的组件。比如Hive之于Hadoop，SparkSQL 之于 Spark。elasticsearch-sql这个项目，提供了类似了SQL翻译功能。



当然，Elasticsearch 最突出的地方在于对于全文搜索的支持。全文搜索的原理，以及在 Elasticsearch中具体如何全文搜索，这里略去不表。下面介绍下 Elasticsearch 集群的实现。

