**Elasticsearch 的fielddata内存控制、预加载以及circuit breaker断路器**

fielddata核心原理

fielddata加载到内存的过程是lazy加载的，对一个analzyed field执行聚合时，才会加载，而且是field-level加载的一个index的一个field，所有doc都会被加载，而不是少数doc不是index-time创建，是query-time创建



fielddata内存限制

elasticsearch.yml： indices.fielddata.cache.size: 20%，超出限制，清除内存已有fielddata数据fielddata占用的内存超出了这个比例的限制，那么就清除掉内存中已有的fielddata数据默认无限制，限制内存使用，但是会导致频繁evict和reload，大量IO性能损耗，以及内存碎片和gc



监控fielddata内存使用



1

2

3

4

5

6

7

8

\#各个分片、索引的fielddata在内存中的占用情况

\[root@elk-node03 ~\]\# curl -X GET 'http://10.0.8.47:9200/\_stats/fielddata?fields=\*'    

 

\#每个node的fielddata在内存中的占用情况

\[root@elk-node03 ~\]\# curl -X GET 'http://10.0.8.47:9200/\_nodes/stats/indices/fielddata?fields=\*'

 

\#每个node中的每个索引的fielddata在内存中的占用情况

\[root@elk-node03 ~\]\# curl -X GET 'http://10.0.8.47:9200/\_nodes/stats/indices/fielddata?level=indices&fields=\*'

circuit breaker断路由

如果一次query load的feilddata超过总内存，就会oom --&gt; 内存溢出;

circuit breaker会估算query要加载的fielddata大小，如果超出总内存，就短路，query直接失败;

在elasticsearch.yml文件中配置如下内容:

indices.breaker.fielddata.limit： fielddata的内存限制，默认60%

indices.breaker.request.limit：  执行聚合的内存限制，默认40%

indices.breaker.total.limit：       综合上面两个，限制在70%以内



限制内存使用 \(Elasticsearch聚合限制内存使用\)



通常为了让聚合\(或者任何需要访问字段值的请求\)能够快点，访问fielddata一定会快点， 这就是为什么加载到内存的原因。但是加载太多的数据到内存会导致垃圾回收\(gc\)缓慢， 因为JVM试着发现堆里面的额外空间，甚至导致OutOfMemory \(即OOM\)异常。



然而让人吃惊的发现, Elaticsearch不是只把符合你的查询的值加载到fielddata. 而是把index里的所document都加载到内存，甚至是不同的 \_type 的document。逻辑是这样的，如果你在这个查询需要访问documents X，Y和Z， 你可能在下一次查询就需要访问别documents。而一次把所有的值都加载并保存在内存 ， 比每次查询都去扫描倒排索引要更方便。



JVM堆是一个有限制的资源需要聪明的使用。有许多现成的机制去限制fielddata对堆内存使用的影响。这些限制非常重要，因为滥用堆将会导致节点的不稳定（多亏缓慢的垃圾回收）或者甚至节点死亡（因为OutOfMemory异常）；但是垃圾回收时间过长，在垃圾回收期间，ES节点的性能就会大打折扣，查询就会非常缓慢，直到最后超时。



如何设置堆大小

对于环境变量 $ES\_HEAP\_SIZE 在设置Elasticsearch堆大小的时候有2个法则可以运用:



1\) 不超过RAM的50%

Lucene很好的利用了文件系统cache，文件系统cache是由内核管理的。如果没有足够的文件系统cache空间，性能就会变差;



2\) 不超过32G

如果堆小于32GB，JVM能够使用压缩的指针，这会节省许多内存：每个指针就会使用4字节而不是8字节。把对内存从32GB增加到34GB将意味着你将有更少的内存可用，因为所有的指针占用了双倍的空间。同样，更大的堆，垃圾回收变得代价更大并且可能导致节点不稳定；这个限制主要是大内存对fielddata影响比较大。



Fielddata大小

参数 indices.fielddata.cache.size 控制有多少堆内存是分配给fielddata。当你执行一个查询需要访问新的字段值的时候，将会把值加载到内存，然后试着把它们加入到fielddata。如果结果的fielddata大小超过指定的大小 ，为了腾出空间，别的值就会被驱逐出去。默认情况下，这个参数设置的是无限制 — Elasticsearch将永远不会把数据从fielddata里替换出去。



这个默认值是故意选择的：fielddata不是临时的cache。它是一个在内存里为了快速执行必须能被访问的数据结构，而且构建它代价非常昂贵。如果你每个请求都要重新加载数据，性能就会很差。



一个有限的大小强迫数据结构去替换数据。下面来看看什么时候去设置下面的值，首先看一个警告: 这个设置是一个保护措施，而不是一个内存不足的解决方案! 



如果你没有足够的内存区保存你的fielddata到内存里，Elasticsearch将会经常性的从磁盘重新加载数据，并且驱逐别的数据区腾出空间。这种数据的驱逐会导致严重的磁盘I/O，并且在内存里产生大量的垃圾，这个会在后面被垃圾回收。



假设你在索引日志，每天使用给一个新的索引。通常情况下你只会对过去1天或者2天的数据感兴趣。即使你把老的索引数据保留着，你也很少查询它们。尽管如此，使用默认的设置， 来自老索引的fielddata也不会被清除出去！fielddata会一直增长直到它触发fielddata circuit breaker --参考断路器--它将阻止你继续加载fielddata。在那个时候你被卡住了。即使你仍然能够执行访问老的索引里的fielddata的查询， 你再也不能加载任何新的值了。相反，我们应该把老的值清除出去给新的值腾出空间。为了防止这种情景，通过在elasticsearch.yml文件里加上如下的配置给fielddata 设置一个上限: indices.fielddata.cache.size:  40%   ,可以设置成堆大小的百分比，也可以是一个具体的值，比如 8gb；通过适当设置这个值，最近被访问的fielddata将被清除出去，给新加载数据腾出空间。

 

在网上可能还会看到另外一个设置参数： indices.fielddata.cache.expire 。千万不要使用这个设置！这个设置高版本已经废弃!!! 这个设置告诉Elasticsearch把比过期时间老的数据从fielddata里驱逐出去，而不管这个值是否被用到。这对性能是非常可怕的 。驱逐数据是有代价的，并且这个有目的的高效的安排驱逐数据并没有任何真正的收获。没有任何理由去使用这个设置!!!! 我们一点也不能从理论上制造一个假设的有用的情景。现阶段存 在只是为了向后兼容。我们在这个书里提到这个设置是因为这个设置曾经在网络上的各种文章里 被作为一个 \`\`性能小窍门'' 被推荐过。记住永远不要使用它!!!!

监控fielddata \(上面提到了\)

监控fielddata使用了多少内存以及是否有数据被驱逐是非常重要的。大量的数据被驱逐会导致严重的资源问题以及不好的性能。



Fielddata使用可以通过下面的方式来监控：

对于单个索引使用 {ref}indices-stats.html\[indices-stats API\]:



1

\[root@elk-node03 ~\]\# curl -X GET 'http://10.0.8.47:9200/\_stats/fielddata?fields=

对于单个节点使用 {ref}cluster-nodes-stats.html\[nodes-stats API\]:



1

\[root@elk-node03 ~\]\# curl -X GET 'http://10.0.8.47:9200/\_nodes/stats/indices/fielddata?fields=\*'

或者甚至单个节点单个索引



1

\[root@elk-node03 ~\]\# curl -X GET 'http://10.0.8.47:9200/\_nodes/stats/indices/fielddata?level=indices&fields=\*'

通过设置 ?fields=\* 内存使用按照每个字段分解了.



断路器\(breaker\)

fielddata的大小是在数据被加载之后才校验的。如果一个查询尝试加载到fielddata的数据比可用的内存大会发生什么情况？答案是不客观的：你将会获得一个OutOfMemory异常。



Elasticsearch包含了一个 fielddata断路器 ，这个就是设计来处理这种情况的。断路器通过检查涉及的字段（它们的类型，基数，大小等等）来估计查询需要的内存。然后检查加 载需要的fielddata会不会导致总的fielddata大小超过设置的堆的百分比。



如果估计的查询大小超过限制，断路器就会触发并且查询会被抛弃返回一个异常。这个发生在数据被加载之前，这就意味着你不会遇到OutOfMemory异常。



Elasticsearch拥有一系列的断路器，所有的这些都是用来保证内存限制不会被突破：

indices.breaker.fielddata.limit

这个 fielddata 断路器限制fielddata的大小为堆大小的60%，默认情况下。



indices.breaker.request.limit

这个 request 断路器估算完成查询的其他部分要求的结构的大小，比如创建一个聚集通， 以及限制它们到堆大小的40%，默认情况下。



indices.breaker.total.limit

这个total断路器封装了 request 和 fielddata 断路器去确保默认情况下这2个 使用的总内存不超过堆大小的70%。



断路器限制可以通过文件 config/elasticsearch.yml 指定，也可以在集群上动态更新:

1

2

3

4

5

curl -PUT 'http://10.0.8.47:9200/\_cluster/settings{

"persistent" : {

"indices.breaker.fielddata.limit" : 40% \(1\)

}

}

这个限制设置的是堆的百分比。



最好把断路器设置成一个相对保守的值。记住fielddata需要和堆共享 request 断路器， 索引内存缓冲区，过滤器缓存，打开的索引的Lucene数据结构，以及各种各样别的临时数据 结构。所以默认为相对保守的60%。过分乐观的设置可能会导致潜在的OOM异常，从而导致整 个节点挂掉。从另一方面来说，一个过分保守的值将会简单的返回一个查询异常，这个异常会被应用处理。 异常总比挂掉好。这些异常也会促使你重新评估你的查询：为什么单个的查询需要超过60%的 堆空间。

断路器和Fielddata大小

在 Fielddata大小部分我们谈到了要给fielddata大小增加一个限制去保证老的不使用 的fielddata被驱逐出去。indices.fielddata.cache.size 和 indices.breaker.fielddata.limit 的关系是非常重要的。如果断路器限制比缓冲区大小要小，就会没有数据会被驱逐。为了能够 让它正确的工作，断路器限制必须比缓冲区大小要大。



我们注意到断路器是和总共的堆大小对比查询大小，而不是和真正已经使用的堆内存区比较。 这样做是有一系列技术原因的（比如，堆可能看起来是满的，但是实际上可能正在等待垃圾 回收，这个很难准确的估算）。但是作为终端用户，这意味着设置必须是保守的，因为它是 和整个堆大小比较，而不是空闲的堆比较。 

