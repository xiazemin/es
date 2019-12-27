**Elasticserach内存优化**

ElasticSearch 自身对内存管理进行了大量优化，但对于持续增长的业务仍需进行一定程度的内存优化（而不是纯粹的添加节点和扩展物理内存），以防止 OOM 发生。ElasticSearch 使用的 JVM 堆中主要包括以下几类内存使用：  
**1.**Segment Memory；  
**2.**Filter Cache；  
**3.**Field Data Cache；  
**4.**Bulk Queue；  
**5.**Indexing Buffer；  
**6.**Cluster State Buffer；  
**7.**超大搜索聚合结果集的 fetch；

**1. 减少 Segment Memory**  
-  删除无用的历史索引。删除办法，使用 rest API

| 12345 | `# 删除指定某个索引DELETE /${INDEX_NAME}# 删除符合 pattern 的某些索引DELETE /${INDEX_PATTERN}` |
| :--- | :--- |


-  关闭无需实时查询的历史索引，文件仍然存在于磁盘，只是释放掉内存，需要的时候可以重新打开。关闭办法，使用 rest API

| 12345 | `# 关闭指定某个索引POST /${INDEX_NAME}/_close# 关闭符合 pattern 的某些索引POST /${INDEX_PATTERN}/_close` |
| :--- | :--- |


-  定期对不再更新的索引做 force merge（会占用大量 IO，建议业务低峰期触发）force merge 办法，使用 rest API

| 12345678910111213 | `# Step1. 在合并前需要对合并速度进行合理限制，默认是 20mb，SSD可以适当放宽到 80mb：PUT/_cluster/settings-d '{"persistent": {"indices.store.throttle.max_bytes_per_sec":"20mb"}}'# Step2. 强制合并 API，示例表示的是最终合并为一个 segment file：# 对某个索引做合并POST /${INDEX_NAME}/_forcemerge?max_num_segments=1# 对某些索引做合并POST /${INDEX_PATTERN}/_forcemerge?max_num_segments=1` |
| :--- | :--- |


**2. Filter Cache**  
默认的 10% heap 设置工作得够好，如果实际使用中 heap 没什么压力的情况下，才考虑加大这个设置。

**3. Field Data Cache**  
对需要排序的字段不进行 analyzed，尽量使用 doc values（5.X版本天然支持，不需要特别设置）。对于不参与搜索的字段 \( fields \)，将其 index 方法设置为 no，如果对分词没有需求，对参与搜索的字段，其 index 方法设置为 not\_analyzed。

**4. Bulk Queue**  
一般来说官方默认的 thread pool 设置已经能很好的工作了，建议不要随意去调优相关的设置，很多时候都是适得其反的效果。

**5. Indexing Buffer**  
这个参数的默认值是10% heap size。根据经验，这个默认值也能够很好的工作，应对很大的索引吞吐量。 但有些用户认为这个 buffer 越大吞吐量越高，因此见过有用户将其设置为 40% 的。到了极端的情况，写入速度很高的时候，40%都被占用，导致OOM。

**6. Cluster State Buffer**  
在超大规模集群的情况下，可以考虑分集群并通过 tribe node 连接做到对用户透明，这样可以保证每个集群里的 state 信息不会膨胀得过大。在单集群情况下，缩减 cluster state buffer 的方法就是减少 shard 数量，shard 数量的确定有以下几条规则：  
**1.**  避免有非常大的分片，因为大分片可能会对集群从故障中恢复的能力产生负面影响。 对于多大的分片没有固定限制，但分片大小为 50GB 通常被界定为适用于各种用例的限制；  
**2.**  尽可能使用基于时间的索引来管理数据。根据保留期（retention period，可以理解成有效期）将数据分组。基于时间的索引还可以轻松地随时间改变主分片和副本分片的数量（以为要生成的下一个索引进行更改）。这简化了适应不断变化的数据量和需求；（周期性的通过删除或者关闭历史索引以减少分片）  
**3.**  小分片会导致小分段\(segment\)，从而增加开销。目的是保持平均分片大小在几GB和几十GB之间。对于具有基于时间数据的用例，通常看到大小在 20GB 和 40GB 之间的分片；  
**4.**  由于每个分片的开销取决于分段数和大小，通过强制操作迫使较小的段合并成较大的段可以减少开销并提高查询性能。一旦没有更多的数据被写入索引，这应该是理想的。请注意，这是一个消耗资源的（昂贵的）操作，较为理想的处理时段应该在非高峰时段执行；\(对应使用 force meger 以减少 segment 数量的优化，目的是降低 segment memory 占用）  
**5.**  可以在集群节点上保存的分片数量与可用的堆内存大小成正比，但这在 Elasticsearch 中没有的固定限制。 一个很好的经验法则是：确保每个节点的分片数量保持在低于每 1GB 堆内存对应集群的分片在 20-25 之间。 因此，具有 32GB 堆内存的节点最多可以有 600-750 个分片；  
**6.**  对于单索引的主分片数，有这么 2 个公式：节点数 &lt;= 主分片数 \*（副本数 + 1） 以及 \(同一索引 shard 数量 \* \(1 + 副本数\)\) &lt; 3 \* 数据节点数，比如有 3 个节点全是数据节点，1 个副本，那么主分片数大于等于 1.5，同时同一索引总分片数需要小于 4.5，因为副本数为 1，所以单节点主分片最适为 2，索引总分片数最适为 6，这样每个节点的总分片为 4；  
**7.**  单分片小于 20GB 的情况下，采用单分片较为合适，请求不存在网络抖动的顾虑；

**小结：**分片不超 20GB，且单节点总分片不超 600。比如互联网区域，每天新建索引\(lw-greenbay-online\) 1 个分片 1 个副本，3 个月前的历史索引都关闭，3 节点总共需要扛 90 \* 2 = 180 个分片，每个分片大约 6 GB，可谓比较健康的状态。

**7. 超大搜索聚合结果集的 fetch**  
避免用户 fetch 超大搜索聚合结果集，确实需要大量拉取数据可以采用 scan & scroll API 来实现。在 ElasticSearch 上搜索数据时，默认只会返回10条文档，当我们想获取更多结果，或者只要结果中的一个区间的数据时，可以通过 size 和 from 来指定。

| 1 | `GET/_search?size=3&from=20` |
| :--- | :--- |


如上的查询语句，会返回排序后的结果中第 20 到第 22 条数据。ElasticSearch 在收到这样的一个请求之后，每一个分片都会返回一个 top22 的搜索结果，然后将这些结果汇总排序，再选出 top22 ，最后取第 20 到第 22 条数据作为结果返回。这样会带来一个问题，当我们搜索的时候，如果想取出第 10001 条数据，那么就相当于每个一分片都要对数据进行排序，取出前 10001 条文档，然后 ElasticSearch 再将这些结果汇总再次排序，之后取出第 10001 条数据。这样对于 ElasticSearch 来说就会产生相当大的资源和性能开销。如果我们不要求 ElasticSearch 对结果进行排序，那么就会消耗很少的资源，所以针对此种情况，ElasticSearch 提供了scan & scroll的搜索方式。

| 12345 | `GET/old_index/_search?search_type=scan&scroll=1m{"query": {"match_all": {}},"size":  1000}` |
| :--- | :--- |


我们可以首先通过如上的请求发起一个搜索，但是这个请求不会返回任何文档，它会返回一个 _\_scroll\_id_ ，接下来我们再通过这个 id 来从 ElasticSearch 中读取数据：

| 12 | `GET/_search/scroll?scroll=1mc2Nhbjs1OzExODpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzExOTpRNV9aY1VyUVM4U0 NMd2pjWlJ3YWlBOzExNjpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzExNzpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzEyMDpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzE7dG90YWxfaGl0czoxOw==` |
| :--- | :--- |


此时除了会返回搜索结果以外，还会再次返回一个 _\_scroll\_id_，当我们下次继续取数据时，需要用最新的 id。

