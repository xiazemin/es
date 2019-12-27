**Elasticserach索引详解**

**-   分词器集成**  
**1.**获取 ES-IKAnalyzer插件：  
地址： https://github.com/medcl/elasticsearch-analysis-ik/releases  
一定要获取匹配的版本

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724173227372-66942044.png)

**2. **安装插件  
将 ik 的压缩包解压到 ES安装目录的plugins/目录下（最好把解出的目录名改一下，防止安装别的插件时同名冲突），然后重启ES。  
**3. **扩展词库  
修改配置文件config/IKAnalyzer.cfg.xml

| 12345678910111213 | `<?xml version="1.0"encoding="UTF-8"?><!DOCTYPE properties SYSTEM"http://java.sun.com/dtd/properties.dtd"><properties><comment>IK Analyzer 扩展配置</comment><!--用户可以在这里配置自己的扩展字典--><entry key="ext_dict">custom/mydict.dic;custom/single_word_low_freq.dic</entry><!--用户可以在这里配置自己的扩展停止词字典--><entry key="ext_stopwords">custom/ext_stopword.dic</entry><!--用户可以在这里配置远程扩展字典--><entry key="remote_ext_dict">location</entry><!--用户可以在这里配置远程扩展停止词字典--><entry key="remote_ext_stopwords">http://xxx.com/xxx.dic</entry></properties>` |
| :--- | :--- |


**4. **测试 IK  
**5. **创建一个索引  
\# curl -XPUT http://localhost:9200/index  
**6.**创建一个映射mapping

| 12345678910 | `# curl -XPOST http://localhost:9200/index/fulltext/_mapping -H 'Content-Type:application/json' -d'{"properties": {"content": {"type":"text","analyzer":"ik_max_word","search_analyzer":"ik_max_word"}}}'` |
| :--- | :--- |


**7. **索引一些文档

| 12345678 | `# curl -XPOST http://localhost:9200/index/fulltext/1 -H 'Content-Type:application/json' -d'{"content":"美国留给伊拉克的是个烂摊子吗"}'# curl -XPOST http://localhost:9200/index/fulltext/2 -H 'Content-Type:application/json' -d'{"content":"公安部：各地校车将享最高路权"}'# curl -XPOST http://localhost:9200/index/fulltext/3 -H 'Content-Type:application/json' -d'{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}'` |
| :--- | :--- |


**8. **搜索

| 1234567891011 | `# curl -XPOST http://localhost:9200/index/fulltext/_search  -H 'Content-Type:application/json' -d'{"query": {"match": {"content":"中国"}},"highlight": {"pre_tags": ["<tag1>","<tag2>"],"post_tags": ["</tag1>","</tag2>"],"fields": {"content": {}}}}'` |
| :--- | :--- |


**-   索引收缩**  
索引的分片数是不可更改的，如要减少分片数可以通过收缩方式收缩为一个新的索引。新索引分片数必须是原分片数的因子值，如原分片数是8，则新索引分片数可以为4、2、1 。

**收缩流程**  
- 先把所有主分片都转移到一台主机上；  
- 在这台主机上创建一个新索引，分片数较小，其他设置和原索引一致；  
- 把原索引的所有分片，复制（或硬链接）到新索引的目录下；  
- 对新索引进行打开操作恢复分片数据；\(可选\)重新把新索引的分片均衡到其他节点上。

**a\) 收缩前准备工作**  
将原索引设置为只读；将原索引各分片的一个副本重分配到同一个节点上，并且要是健康绿色状态。

| 1234567 | `PUT/my_source_index/_settings{"settings": {"index.routing.allocation.require._name":"shrink_node_name","index.blocks.write":true}}` |
| :--- | :--- |


**b\) 进行收缩**

| 1234567 | `POST my_source_index/_shrink/my_target_index{"settings": {"index.number_of_replicas": 1,"index.number_of_shards": 1,"index.codec":"best_compression"}}` |
| :--- | :--- |


**c\) 监控收缩过程**

| 12 | `GET _cat/recovery?vGET _cluster/health` |
| :--- | :--- |


**-   Split Index 拆分索引**  
当索引的分片容量过大时，可以通过拆分操作将索引拆分为一个倍数分片数的新索引。能拆分为几倍由创建索引时指定的index.number\_of\_routing\_shards 路由分片数决定。这个路由分片数决定了根据一致性hash路由文档到分片的散列空间。如index.number\_of\_routing\_shards = 30 ，指定的分片数是5，则可按如下倍数方式进行拆分：  
5 → 10 → 30 \(split by 2, then by 3\)  
5 → 15 → 30 \(split by 3, then by 2\)  
5 → 30 \(split by 6\)  
**注意：**只有在创建时指定了index.number\_of\_routing\_shards 的索引才可以进行拆分，ES7开始将不再有这个限制。  
**a\) 准备一个索引来做拆分**

| 1234567 | `PUT my_source_index{"settings": {"index.number_of_shards": 1,"index.number_of_routing_shards": 2 }}` |
| :--- | :--- |


**b\) 设置索引只读**

| 123456 | `PUT/my_source_index/_settings{"settings": {"index.blocks.write":true}}` |
| :--- | :--- |


**c\) 监控拆分过程**

| 12 | `GET _cat/recovery?vGET _cluster/health` |
| :--- | :--- |


**-   别名滚动**  
对于有时效性的索引数据，如日志，过一定时间后，老的索引数据就没有用了。我们可以像数据库中根据时间创建表来存放不同时段的数据一样，在ES中也可用建多个索引的方式来分开存放不同时段的数据。比数据库中更方便的是ES中可以通过别名滚动指向最新的索引的方式，让你通过别名来操作时总是操作的最新的索引。

ES的rollover index API 让我们可以根据满足指定的条件（时间、文档数量、索引大小）创建新的索引，并把别名滚动指向新的索引。  
-  Rollover Index 示例  
-  创建一个名字为logs-0000001 、别名为logs\_write 的索引

| 123456 | `PUT/logs-000001{"aliases": {"logs_write": {}}}` |
| :--- | :--- |


-  如果别名logs\_write指向的索引是7天前（含）创建的或索引的文档数&gt;=1000或索引的大小&gt;= 5gb，则会创建一个新索引 logs-000002，并把别名logs\_writer指向新创建的logs-000002索引

| 12345678910 | `# Add > 1000 documents to logs-000001POST/logs_write/_rollover{"conditions": {"max_age": "7d","max_docs":  1000,"max_size":"5gb"}}` |
| :--- | :--- |


-  Rollover Index 新建索引的命名规则  
如果索引的名称是-数字结尾，如logs-000001，则新建索引的名称也会是这个模式，数值增1。  
如果索引的名称不是-数值结尾，则在请求rollover api时需指定新索引的名称:

| 12345678 | `POST/my_alias/_rollover/my_new_index_name{"conditions": {"max_age": "7d","max_docs":  1000,"max_size":"5gb"}}` |
| :--- | :--- |


-  在名称中使用Date math（时间表达式）  
如果你希望生成的索引名称中带有日期，如logstash-2016.02.03-1 ，则可以在创建索引时采用时间表达式来命名:

| 123456789101112131415161718192021222324 | `# PUT /<logs-{now/d}-1> with URIencoding:PUT /%3Clogs-%7Bnow%2Fd%7D-1%3E{"aliases": {"logs_write": {}}}PUT logs_write/_doc/1{"message":"a dummy log"}POST logs_write/_refresh# Wait for a day to passPOST/logs_write/_rollover{"conditions": {"max_docs": "1"}}` |
| :--- | :--- |


**注意：**rollover是你请求它才会进行操作，并不是自动在后台进行的。你可以周期性地去请求它。

**-   路由**  
**1. 集群组成**

| 123456789101112 | `集群元信息Cluster-name：essNodes:node1   10.0.1.11   masternode2   10.0.1.12node3   10.0.1.13Indics:s0:shard0:primay: 10.0.1.11rep:10.0.1.13……` |
| :--- | :--- |


![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724185219173-1866658991.png)

**2. 创建索引的流程**

| 123456789 | `PUT s1{"settings": {"index": {"number_of_shards": 3,"number_of_replicas": 1   }}}` |
| :--- | :--- |


**1.**请求node3创建索引  
**2.**node3请求转发给master节点  
**3.**选择节点存放分片、副本，记录元信息  
**4.**通知给参与存放索引分片、副本的节点从节点，创建分片、副本  
**5.**参与节点向主节点反馈结果  
**6.**等待时间到了，master向node3反馈结果信息，node3响应请求。  
**7.**主节点将元信息广播给所有从节点。

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724185407114-1895766867.png)

**3. 节点故障**

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724185530452-1716728401.png)

| 123456789101112 | `集群元信息Cluster-name：essNodes:node1  10.0.1.11   master 故障node2   10.0.1.12   masternode3   10.0.1.13 Indics:s0:shard0:primay:10.0.1.12rep:10.0.1.13……` |
| :--- | :--- |


节点数据自动重新分配

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724185641266-1608388687.png)

**4. 索引文档**  
索引文档的步骤：  
**1.**node2计算文档的路由值得到文档存放的分片（假定路由选定的是分片0）。  
**2.**将文档转发给分片0的主分片节点 node1。  
**3. **node1索引文档，同步给副本节点node3索引文档。  
**4.**node1向node2反馈结果  
**5.**node2作出响应

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724190335009-60869552.png)

**5. 搜索**  
**1.**node2解析查询。  
**2.**node2将查询发给索引s1的分片/副本（R1,R2,R0）节点  
**3.**各节点执行查询，将结果发给Node2  
**4.**Node2合并结果，作出响应。

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724191201846-1944678889.png)

**6. 文档如何路由**  
**1.**文档该存到哪个分片上？  
决定文档存放到哪个分片上就是文档路由。ES中通过下面的计算得到每个文档的存放分片：  
shard = hash\(routing\) % number\_of\_primary\_shards  
routing 是用来进行hash计算的路由值，默认是使用文档id值。我们可以在索引文档时通过routing参数指定别的路由值，在索引、删除、更新、查询中都可以使用routing参数（可多值）指定操作的分片。

| 1234567891011121314151617 | `POST twitter/_doc?routing=kimchy{"user":"kimchy","post_date":"2009-11-15T14:12:12","message":"trying out Elasticsearch"}强制要求给定路由值PUT my_index2{"mappings": {"_doc": {"_routing": {"required":true}}}}` |
| :--- | :--- |


**2.**关系型数据库中有分区表，通过选定分区，可以降低操作的数据量，提高效率。在ES的索引中能不能这样做？  
可以：通过指定路由值，让一个分片上存放一个区的数据。如按部门存放数据，则可指定路由值为部门值。

