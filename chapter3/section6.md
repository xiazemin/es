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



