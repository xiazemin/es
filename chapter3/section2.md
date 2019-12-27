# Elasticsearch性能优化



Elasticsearch是响应如前所述大多数用例的最热门的开源数据存储引擎之一。

Elasticsearch是一种分布式数据存储和搜索引擎，具有容错和高可用性特点。为了充分利用其搜索功能，需要正确配置Elasticsearch。

简单的默认配置不适合每个实际业务场景。实战开发运维中，个性化实现贴合自己业务场景的集群配置是优化集群性能的必经之路。本文集合实战业务场景，重点介绍搜索密集型Elasticsearch集群的提升性能的干货配置。



1、索引层面优化配置

默认情况下，6.x及之前的版本中Elasticsearch索引有5个主分片和1个副本，7.X及之后版本1主1副。 这种配置并不适用于所有业务场景。 需要正确设置分片配置，以便维持索引的稳定性和有效性。

1.1、分片大小

分片大小对于搜索查询非常重要。

一方面， 如果分配给索引的分片太多，则Lucene分段会很小，从而导致开销增加。当同时进行多个查询时，许多小分片也会降低查询吞吐量。

另一方面，太大的分片会导致搜索性能下降和故障恢复时间更长。

Elasticsearch官方建议一个分片的大小应该在20到40 GB左右。

例如，如果您计算出索引将存储300 GB的数据，则可以为该索引分配9到15个主分片。

根据集群大小，假设群集中有10个节点，您可以选择为此索引分配10个主分片，以便在集群节点之间均匀分配分片。

1.2、数据动态持续写入场景

如果存在连续写入到Elasticsearch集群的数据流，如：实时爬虫互联网数据写入ES集群。则应使用基于时间的索引以便更轻松地维护索引。

如果写入数据流的吞吐量随时间而变化，则需要适当地改变下一个索引的配置才能实现数据的动态扩展。

那么，如何查询分散到不同的基于时间索引的所有文档？答案是别名。可以将多个索引放入别名中，并且对该别名进行搜索会使查询就像在单个索引上一样。

当然，需要保持好平衡。注意思考：将多少数据写入别名？别名上写入太多小索引会对性能产生负面影响。

例如，是以周还是以月为单位为单位建立索引是需要结合业务场景平衡考虑的问题？

如果以月为单位建议索引性能最优，那么相同数据以周为单位建立索引势必会因为索引太多导致负面的性能问题。

1.3、Index Sorting

注意：索引排序机制是6.X版本才有的特性。

在Elasticsearch中创建新索引时，可以配置每个分片中的分段的排序方式。 默认情况下，Lucene不会应用任何排序。 index.sort.\* 定义应使用哪些字段对每个Segment内的文档进行排序。

使用举例：



PUT /twitter

 {

     "settings" : {

         "index" : {

             "sort.field" : "date", 

             "sort.order" : "desc" 

         }

     },

     "mappings": {

        "properties": {

            "date": {

                "type": "date"

            }

        }

    }

}

目的：index sorting是优化Elasticsearch检索性能的非常重要的方式之一。

大白话：index sorting机制通过写入的时候指定了某一个或者多个字段的排序方式，会极大提升检索的性能。



2、分片层面优化配置

分片是底层基本的读写单元，分片的目的是分割巨大索引，让读写并行执行。写入过程先写入主分片，主分片写入成功后再写入副本分片。

副本分片的出现，提升了集群的高可用性和读取吞吐率。

在优化分片时，分片的大小、节点中有多少分片是主要考虑因素。副本分片对于扩展搜索吞吐量很重要，如果硬件条件允许，则可以小心增加副本分片的数量。

容量规划的一个很好的启动点是分配分片，“《深入理解Elasticsearch》强调：最理想的分片数量应该依赖于节点的数量。”其数量是节点数量的1.5到3倍。

分配副本分片数的公式：max（max\_failures，ceil（num\_nodes /） num\_primaries） - 1）。

原理：如果您的群集具有num\_nodes节点，总共有num\_primaries主分片，如果您希望最多能够同时处理max\_failures节点故障，那么适合您的副本数量为如上公式值。

总的来说：节点数和分片数、副本数的简单计算公式如下：

所需做大节点数=分片数\*（副本数+1）。



3、Elasticsearch整体层面配置

配置Elasticsearch集群时，最主要的考虑因素之一是确保至少有一半的可用内存进入文件系统缓存，以便Elasticsearch可以将索引的hot regions保留在物理内存中。

在设计集群时还应考虑物理可用堆空间。 Elasticsearch建议基于可用堆空间的分片分配最多应为20个分片/ GB，这是一个很好的经验法则。

例如，具有30 GB堆的节点最多应有600个分片，以保持集群的良好状态。

一个节点上的存储可以表述如下：节点可以支持的磁盘空间= 20 （堆大小单位：GB）（以GB为单位的分片大小），由于在高效集群中通常会看到大小在20到40 GB之间的分片，因此最大存储空间可以支持16 GB可用堆空间的节点，最多可达12 TB的磁盘空间（201640=12.8TB）。

边界意识有助于为更好的设计和未来的扩展操作做好准备。

可以在运行时以及初始阶段进行许多配置设置。

在构建Elasticsearch索引和集群本身以获得更好的搜索性能时，了解在运行时哪些配置可以修改以及哪些配不可以修改是至关重要的。

3.1 动态设置

1、设置历史数据索引为只读状态。

基于时间的动态索引的执行阶段，如果存放历史数据的索引没有写操作，可以将月度索引设置为只读模式，以提高对这些索引的搜索性能。

6.X之后的只读索引实战设置方式：



PUT /twitter/\_settings

{

  "index.blocks.read\_only\_allow\_delete": null

}

2、对只读状态索引，进行段合并。

当索引设置为只读时，可以通过强制段合并操作以减少段的数量。

优化段合并将导致更好的搜索性能，因为每个分片的开销取决于段的计数和大小。

注意1：不要将段合并用于读写索引，因为它将导致产生非常大的段（每段&gt; 5Gb）。

注意2：此操作应在非高峰时间进行，因为这是一项非常耗资源的操作。

段合并操作实战方式：

curl -X POST "localhost:9200/kimchy/\_forcemerge?only\_expunge\_deletes=false&max\_num\_segments=100&flush=true"



3、使用preference优化缓存利用率

有多个缓存可以帮助提高搜索性能，例如文件系统缓存，请求缓存或查询缓存。

然而，所有这些缓存都维护在节点级别，这意味着如果您在拥有1个或更多副本且基于默认路由算法集群上连续两次运行相同的请求，这两个请求将转到不同的分片副本上 ，阻止节点级缓存帮助。

由于搜索应用程序的用户一个接一个地运行类似的请求是常见的，例如为了检索分析索引的部分较窄子集，使用preference标识当前用户或会话的偏好值可以帮助优化高速缓存的使用。

preference实战举例：



GET /\_search?preference=xyzabc123

{

    "query": {

        "match": {

            "title": "elasticsearch"

        }

    }

4、禁止交换

可以在每个节点上禁用交换以确保稳定性，并且应该不惜一切代价避免交换。它可能导致垃圾收集持续数分钟而不是毫秒，并且可能导致节点响应缓慢甚至断开与集群的连接。

在Elasticsearch分布式系统中，让操作系统终止节点更有效。可以通过将bootstrap.memory\_lock设置为True来禁用它。

Linux系统级配置：

\`sudo swapoff -a\`\`



Elasticsearch配置文件elasticsearch.yml配置：

bootstrap.memory\_lock: true



5、增加刷新间隔 refresh\_interval

默认刷新间隔为1秒。这迫使Elasticsearch每秒创建一个分段。实际业务中，应该根据使用情况增加刷新间隔，举例：增加到30秒。

这样之后，30s产生一个大的段，较每秒刷新大大减少未来的段合并压力。最终会提升写入性能并使搜索查询更加稳定。

更新刷新间隔实战：



PUT /twitter/\_settings

{

    "index" : {

        "refresh\_interval" : "1s"

    }

}

6、设置max\_thread\_count

index.merge.scheduler.max\_thread\_count默认设置为

Math.max\(1, Math.min\(4, Runtime.getRuntime\(\).availableProcessors\(\) / 2\)\)

但这适用于SSD配置。对于HDD，应将其设置为1。

实战：



curl -XPUT 'localhost:9200/\_settings' -d '{ 

     "index.merge.scheduler.max\_thread\_count" : 1

}

7、禁止动态分配分片

有时，Elasticsearch将重新平衡集群中的分片。此操作可能会降低检索的性能。

在生产模式下，需要时，可以通过cluster.routing.rebalance.enable设置将重新平衡设置为none。

PUT /\_cluster/settings



{ 

  "transient" : {

    "cluster.routing.allocation.enable" : "none"

  }

}

其中典型的应用场景之包括：

集群中临时重启、剔除一个节点；

集群逐个升级节点；当您关闭节点时，分配过程将立即尝试将该节点上的分片复制到集群中的其他节点，从而导致大量浪费的IO. 在关闭节点之前禁用分配可以避免这种情况。



8、充分利用近似日期缓存效果

现在使用的日期字段上的查询通常不可缓存，因为匹配的范围一直在变化。

然而，就用户体验而言，切换到近似日期通常是可接受的，并且能更好地使用查询高速缓存带来的益处。

实战如下：



 GET index/\_search

 {

   "query": {

     "constant\_score": {

       "filter": {

         "range": {

           "my\_date": {

             "gte": "now-1h/m",

             "lte": "now/m"

          }

        }

      }

    }

  }

}

3.2 初始设置

1、合并多字段提升检索性能

query\_string或multi\_match查询所针对的字段越多，检索越慢。

提高多个字段的搜索速度的常用技术是在索引时将其值复制到单个字段中。

对于经常查询的某些字段，请使用Elasticsearch的copy-to功能。

例如，汽车的品牌名称，发动机版本，型号名称和颜色字段可以与复制到指令合并。它将改善在这些字段上进行的搜索查询性能。



PUT movies

 {

   "mappings": {

     "properties": {

       "cars\_infos": {

         "type": "text"

       },

       "brand\_name": {

         "type": "text",

        "copy\_to": "cars\_infos"

      },

      "engine\_version": {

        "type": "text",

        "copy\_to": "cars\_infos"

      },

   "model ": {

        "type": "text",

        "copy\_to": "cars\_infos"

      },

   "color": {

        "type": "text",

        "copy\_to": "cars\_infos"

      }

    }

  }

}

2、设置分片分配到指定节点

实战业务中经常遇到的业务场景问题：如何将分片设置非均衡分配，有新节点配置极高，能否多分片点过去？

某个 shard 分配在哪个节点上，一般来说，是由 ES 自动决定的。以下几种情况会触发分配动作：

1）新索引生成

2）索引的删除

3）新增副本分片

4）节点增减引发的数据均衡

ES 提供了一系列参数详细控制这部分逻辑，其中之一是：在异构集群的情为具有更好硬件的节点的分片分配分配权重。

为了分配权重，

需要设置cluster.routing.allocation.balance.shard值，默认值为0.45f。

数值越大越倾向于在节点层面均衡分片。

实战：



PUT \_cluster/settings

{

“transient” : {

“cluster.routing.allocation.balance.shard” : 0.60

}

}

3、调整熔断内存比例大小

查询本身也会对响应的延迟产生重大影响。为了在查询时不触发熔断并导致Elasticsearch集群处于不稳定状态，

可以根据查询的复杂性将indices.breaker.total.limit设置为适合您的JVM堆大小。此设置的默认值是JVM堆的70％。



PUT /\_cluster/settings

{

  "persistent" : {

    "indices.breaker.fielddata.limit" : "60%" 

  }

}

最好为断路器设置一个相对保守点的值。

《Elastic源码分析》作者张超指出：“Elasticsearch 7.0 增加了 indices.breaker.total.use\_real\_memory 配置项，可以更加精准的分析当前的内存情况，及时防止 OOM 出现。虽然该配置会增加一点性能损耗，但是可以提高 JVM 的内存使用率，增强了节点的保护机制。”

4、特定搜索场景，增加搜索线程池配置

默认情况下，Elasticsearch将主要用例是搜索。在需要增加检索并发性的情况下，可以增加用于搜索设置的线程池，与此同时，可以根据节点上的CPU中的核心数量多少斟酌减少用于索引的线程池。

举例：更改配置文件elasticsearch.yml增加如下内容：



thread\_pool.search.queue\_size: 500

\#queue\_size允许控制没有线程执行它们的挂起请求队列的初始大小。

5、打开自适应副本选择

应打开自适应副本选择。该请求将被重定向到响应最快的节点。

当存在多个数据副本时，elasticsearch可以使用一组称为自适应副本选择的标准，根据包含每个分片副本的节点的响应时间，服务时间和队列大小来选择数据的最佳副本。

这样可以提高查询吞吐量并减少搜索量大的应用程序的延迟。

这个配置默认是关闭的，实战打开方法：



PUT /\_cluster/settings

{

    "transient": {

        "cluster.routing.use\_adaptive\_replica\_selection": true

    }

}

