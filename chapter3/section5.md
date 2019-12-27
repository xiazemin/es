# 如何物理删除给定期限的历史数据

想到删除，基础认知是delete，细分为删除文档（document）和删除索引；要删除历史数据，基础认知是：删除了给定条件的数据，用delete\_by\_query。

实际操作发现：



删除文档后，磁盘空间并没有立即减少，反而增加了？

除了定时任务+delete\_by\_query，有没有更好的方式呢？



2、常见的删除操作

2.1 删除单个文档

DELETE /twitter/\_doc/1

2.2 删除满足给定条件的文档

POST twitter/\_delete\_by\_query

{

  "query": { 

    "match": {

      "message": "some message"

    }

  }

}

注意：执行批量删除的时候，可能会发生版本冲突。强制执行删除的方式如下：



POST twitter/\_doc/\_delete\_by\_query?conflicts=proceed

{

  "query": {

    "match\_all": {}

  }

}

2.3 删除单个索引

DELETE /twitter

2.4 删除所有索引

DELETE /\_all

或者



DELETE /\*

删除所有索引是非常危险的操作，要注意谨慎操作。



3、删除文档后台做了什么？

执行删除后的返回结果：



{

  "\_index": "test\_index",

  "\_type": "test\_type",

  "\_id": "22",

  "\_version": 2,

  "result": "deleted",

  "\_shards": {

    "total": 2,

    "successful": 1,

    "failed": 0

  },

  "\_seq\_no": 2,

  "\_primary\_term": 17

}

解读：



索引的每个文档都是版本化的。

删除文档时，可以指定版本以确保我们试图删除的相关文档实际上被删除，并且在此期间没有更改。



每个在文档上执行的写操作，包括删除，都会使其版本增加。



真正的删除时机：



deleting a document doesn’t immediately remove the document from disk; it just marks it as &gt;deleted. Elasticsearch will clean up deleted documents in the background as you continue to index more data.



4、删除索引和删除文档的区别？

1）删除索引是会立即释放空间的，不存在所谓的“标记”逻辑。



2）删除文档的时候，是将新文档写入，同时将旧文档标记为已删除。 磁盘空间是否释放取决于新旧文档是否在同一个segment file里面，因此ES后台的segment merge在合并segment file的过程中有可能触发旧文档的物理删除。



但因为一个shard可能会有上百个segment file，还是有很大几率新旧文档存在于不同的segment里而无法物理删除。想要手动释放空间，只能是定期做一下force merge，并且将max\_num\_segments设置为1。



POST /\_forcemerge

5、如何仅保存最近100天的数据？

有了上面的认知，仅保存近100天的数据任务分解为：



1）delete\_by\_query设置检索近100天数据；

2）执行forcemerge操作，手动释放磁盘空间。

删除脚本如下：



\#!/bin/sh

curl -H'Content-Type:application/json' -d'{

    "query": {

        "range": {

            "pt": {

                "lt": "now-100d",

                "format": "epoch\_millis"

            }

        }

    }

}

' -XPOST "http://192.168.1.101:9200/logstash\_\*/

\_delete\_by\_query?conflicts=proceed"

merge脚本如下：



\#!/bin/sh

curl -XPOST 'http://192.168.1.101:9200/\_forcemerge?

only\_expunge\_deletes=true&max\_num\_segments=1'

6、有没有更通用的方法？

有，使用ES官网工具——curator工具。



6.1 curator简介

主要目的：规划和管理ES的索引。支持常见操作：创建、删除、合并、reindex、快照等操作。



6.2 curator官网地址

http://t.cn/RuwN0oM



Git地址：https://github.com/elastic/curator



6.3 curator安装向导

地址：http://t.cn/RuwCkBD



注意：

curator各种博客教程层出不穷，但curator旧版本和新版本有较大差异，建议参考官网最新手册部署。

旧版本命令行方式新版本已不支持。



6.4 curator命令行操作

$ curator --help

Usage: curator \[OPTIONS\] ACTION\_FILE



  Curator for Elasticsearch indices.



  See http://elastic.co/guide/en/elasticsearch/client/curator/current



Options:

  --config PATH  Path to configuration file. Default: ~/.curator/curator.yml

  --dry-run      Do not perform any changes.

  --version      Show the version and exit.

  --help         Show this message and exit.

核心：

--配置文件config.yml：配置要连接的ES地址、日志配置、日志级别等；



执行文件action.yml: 配置要执行的操作\(可批量）、配置索引的格式（前缀匹配、正则匹配方式等）

6.5 curator适用场景

最重要的是：



仅以删除操作为例：curator可以非常简单地删除x天后的索引的前提是：索引命名要遵循特定的命名模式——如:以天为命名的索引：logstash\_2018.04.05。

命名模式需要和action.yml中的delete\_indices下的timestring对应。

