**志查询工具**

TalkingData 移动广告监测产品Ad Tracking（简称ADT）的系统会接收媒体发过来的点击数据以及SDK发过来的激活和各种效果点数据，这些数据的处理过程正确与否至关重要。例如，设备的一条激活数据为啥没有归因到点击，这类问题的排查在Ad Tracking中很常见，通过将数据流中的各个处理环节的重要日志统一发送到ES，可以很方便的进行查询，技术支持的同事可以通过拼写简单的查询条件排查客户的问题。

* 索引按天创建：定时关闭历史索引，释放集群资源
* 别名查询：数据量增大之后，可以通过拆分索引减轻写入压力，拆分之后的索引采用相同的别名，查询服务不需要修改代码
* 索引重要的设置：

```
{
    "settings": {
        "index": {
            "refresh_interval": "120s",
            "number_of_shards": "12",
            "translog": {
                "flush_threshold_size": "2048mb"
            },
            "merge": {
                "scheduler": {
                    "max_thread_count": "1"
                }
            },
            "unassigned": {
                "node_left": {
                    "delayed_timeout": "180m"
                }
            }
        }
    }
}

```

* 索引mapping的设置

```
{
    "properties": {
        "action_content": {
            "type": "string",
            "analyzer": "standard"
        },
        "time": {
            "type": "long"
        },
        "trackid": {
            "type": "string",
            "index": "not_analyzed"
        }
    }
}

```

* sql插件，通过拼sql的方式，比起拼json更简单

**点击数据存储（kv存储场景）**

Ad Tracking收集的点击数据是与广告投放直接相关的数据，应用安装之后，SDK会上报激活事件，系统会去查找这个激活事件是否来自于之前用户点击的某个广告，如果是，那么该激活就是一个推广量，也就是投放的广告带来的激活。激活后续的效果点数据也都会去查找点击，从点击中获取广告投放的一些信息，所以点击查询在Ad Tracking的业务中至关重要。

业务的前期，点击数据是存储在Mysql中的，随着后续点击量的暴增，由于Mysql不能横向扩展，所以需要更换为新的存储。由于ES拥有横向扩展和强悍的搜索能力，并且之前日志查询工具中也一直使用ES，所以决定使用ES来进行点击的存储。

重要的设置

* "refresh\_interval": "1s"
* "translog.flush\_threshold\_size": "2048mb"
* "merge.scheduler.max\_thread\_count": 1
* "unassigned.node\_left.delayed\_timeout": "180m"

结合业务进行系统优化

结合业务定期关闭索引释放资源：Ad Tracking的点击数据具有有效期的概念，超过有效期的点击，激活不会去归因。点击有效期最长一个月，所以理论上每天创建的索引在一个月之后才能关闭。但是用户配置的点击有效期大部分都是一天，这大部分点击在集群中保存30天是没有意义的，而且会占用大部分的系统资源。所以根据点击的这个业务特点，将每天创建的索引拆分成两个，一个是有效期是一天的点击，一个是超过一天的点击，有效期一天的点击的索引在一天之后就可以关闭，从而保证集群中打开的索引的数据量维持在一个较少的水平。

结合业务将热点数据单独索引：激活和效果点数据都需要去ES中查询点击，但是两者对于点击的查询场景是有差异的，因为效果点事件（例如登录、注册等）归因的时候不是去直接查找点击，而是查找激活进而找到点击，效果点要找的点击一定是之前激活归因到的，所以激活归因到的这部分点击也就是热点数据。激活归因到点击之后，将这部分点击单独存储到单独的索引中，由于这部分点击量少很多，所以效果点查询的时候很快。

索引拆分：Ad Tracking的点击数据按天进行存储，但是随着点击量的增大，单天的索引大小持续增大，尤其是晚上的时候，merge需要合并的segment数量以及大小都很大，造成了很高的IO压力，导致集群的写入受限。后续采用了拆分索引的方案，每天的索引按照上午9点和下午5点两个时间点将索引拆分成三个，由于索引之间的segment合并是相互独立的，只会在一个索引内部进行segment的合并，所以在每个小索引内部，segment合并的压力就会减少。

**其他调优**

分片的数量

经验值：

* 每个节点的分片数量保持在低于每1GB堆内存对应集群的分片在20-25之间。
* 分片大小为50GB通常被界定为适用于各种用例的限制。

JVM设置

* 堆内存设置：不要超过32G，在Java中，对象实例都分配在堆上，并通过一个指针进行引用。对于64位操作系统而言，默认使用64位指针，指针本身对于空间的占用很大，Java使用一个叫作内存指针压缩（compressed oops）的技术来解决这个问题，简单理解，使用32位指针也可以对对象进行引用，但是一旦堆内存超过32G，这个压缩技术不再生效，实际上失去了更多的内存。
* 预留一半内存空间给lucene用，lucene会使用大量的堆外内存空间。
* 如果你有一台128G的机器，一半内存也是64G，超过了32G，可以通过一台机器上启动多个ES实例来保证ES的堆内存小于32G。
* ES的配置文件中加入bootstrap.mlockall: true，关闭内存交换。

通过\_cat api获取任务执行情况

GET http://localhost:9201/\_cat/thread\_pool?v&h=host,search.active,search.rejected,search.completed

* 完成\(completed\)
* 进行中\(active\)
* 被拒绝\(rejected\)：需要特别注意，说明已经出现查询请求被拒绝的情况，可能是线程池大小配置的太小，也可能是集群性能瓶颈，需要扩容。

小技巧

* 重建索引或者批量想ES写历史数据的时候，写之前先关闭副本，写入完成之后，再开启副本。
* ES默认用文档id进行路由，所以通过文档id进行查询会更快，因为能直接定位到文档所在的分片，否则需要查询所有的分片。
* 使用ES自己生成的文档id写入更快，因为ES不需要验证一次自定义的文档id是否存在。

参考资料

[https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html](https://link.zhihu.com/?target=https%3A//www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

[https://github.com/Neway6655/neway6655.github.com/blob/master/\_posts/2015-09-11-elasticsearch-study-notes.md](https://link.zhihu.com/?target=https%3A//github.com/Neway6655/neway6655.github.com/blob/master/_posts/2015-09-11-elasticsearch-study-notes.md)

[https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps](https://link.zhihu.com/?target=https%3A//www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps)

[https://blog.csdn.net/zteny/article/details/85245967](https://link.zhihu.com/?target=https%3A//blog.csdn.net/zteny/article/details/85245967)

  


