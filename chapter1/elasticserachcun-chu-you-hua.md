Elasticserach存储优化

1. 关闭不需要的功能

默认情况下 ElasticSearch 会将 indexs 和 doc values 添加到大多数字段中，以便可以搜索和聚合它们。 例如，如果有一个名为 foo 的数字字段，需要运行 histograms 但不需要 filter，则可以安全地禁用映射中此字段的索引：

PUT ${INDEX\_NAME}

{

"mappings": {

```
"type": {

  "properties": {

    "foo": {

      "type": "integer",

      "index": false

    }

  }

}
```

}

}

text 字段在索引中存储规范化因子以便能够对文档进行评分。 如果只需要在 text 字段上使用 matching 功能，但不关心生成的 score，则可以命令 ElasticSearch 配置为不将规范写入索引：

PUT ${INDEX\_NAME}

{

"mappings": {

```
"type": {

  "properties": {

    "foo": {

      "type": "text",

      "norms": false

    }

  }

}
```

}

}

text 字段也默认存储索引中的频率和位置。 频率用于计算分数，位置用于运行短语查询（phrase queries）。 如果不需要运行短语查询，可以告诉 ElasticSearch 不要索引位置：

PUT ${INDEX\_NAME}

{

"mappings": {

```
"type": {

  "properties": {

    "foo": {

      "type": "text",

      "index\_options": "freqs"

    }

  }

}
```

}

}

此外，如果不关心计分，则可以配置 ElasticSearch 以仅索引每个 term 的匹配文档。 这样做仍然可以在此字段上进行搜索\(search\)，但是短语查询会引发错误，评分将假定 term 在每个文档中只出现一次。

PUT ${INDEX\_NAME}

{

"mappings": {

```
"type": {

  "properties": {

    "foo": {

      "type": "text",

      "norms": false,

      "index\_options": "freqs"

    }

  }

}
```

}

}

1. 强制清除已标记删除的数据

Elasticsearch 是建立在 Apache Lucene 基础上的实时分布式搜索引擎，Lucene 为了提高搜索的实时性，采用不可再修改（immutable）方式将文档存储在一个个 segment 中。也就是说，一个 segment 在写入到存储系统之后，将不可以再修改。那么 Lucene 是如何从一个 segment 中删除一个被索引的文档呢？简单的讲，当用户发出命令删除一个被索引的文档\#ABC 时，该文档并不会被马上从相应的存储它的 segment 中删除掉，而是通过一个特殊的文件来标记该文档已被删除。当用户再次搜索到 \#ABC 时，Elasticsearch 在 segment 中仍能找到 \#ABC，但由于 \#ABC 文档已经被标记为删除，所以Lucene 会从发回给用户的搜索结果中剔除 \#ABC，所以给用户感觉的是 \#ABC 已经被删除了。

Elasticseach 会有后台线程根据 Lucene 的合并规则定期进行 segment merging 合并操作，一般不需要用户担心或者采取任何行动。被删除的文档在 segment 合并时，才会被真正删除掉。在此之前，它仍然会占用着JVM heap和操作系统的文件cach 等资源。在某些情况下，需要强制 Elasticsearch 进行 segment merging，已释放其占用的大量系统资源。

POST /${INDEX\_NAME}/\_forcemerge?max\_num\_segments=1&only\_expunge\_deletes=true&wait\_for\_completion=true

POST /${INDEX\_PATTERN}/\_forcemerge?max\_num\_segments=1&only\_expunge\_deletes=true&wait\_for\_completion=true

Force Merge 命令可强制进行 segment 合并，并删除所有标记为删除的文档。Segment merging 要消耗 CPU，以及大量的 I/O 资源，所以一定要在 ElasticSearch 集群处于维护窗口期间，并且有足够的 I/O 空间的（如：SSD）的条件下进行；否则很可能造成集群崩溃和数据丢失。

1. 减少副本数

最直接的存储优化手段是调整副本数，默认 ElasticSearch 是有 1 个副本的，假设对可用性要求不高，允许磁盘损坏情况下可能的数据缺失，可以把副本数调整为0，操作如下：

PUT  /\_template/${TEMPLATE\_NAME}

{

"template":"${TEMPLATE\_PATTERN}",

"settings" : {

```
"number\_of\_replicas" : 0
```

},

"version"  : 1

}

其中 ${TEMPLATE\_NAME} 表示模板名称，可以是不存在的，系统会新建。${TEMPLATE\_PATTERN} 是用于匹配索引的表达式，比如 lw-greenbay-online-\*。

与此相关的一个系统参数为：index.merge.scheduler.max\_thread\_count，默认值为 Math.max\(1, Math.min\(4, Runtime.getRuntime\(\).availableProcessors\(\) / 2\)\)，这个值在 SSD 上工作没问题，但是 SATA 盘上还是使用 1 个线程为好，因为太多也来不及完成。

\# SATA 请设置 merge 线程为 1

PUT  /\_template/${TEMPLATE\_NAME}

{

"template":"${TEMPLATE\_PATTERN}",

"settings" : {

```
"index.merge.scheduler.max\_thread\_count": 1
```

},

"version"  : 1

}

1. 请勿使用默认的动态字符串映射

默认的动态字符串映射会将字符串字段索引为文本\(text\)和关键字\(keyword\)。 如果只需要其中的一个，这样做无疑是浪费的。 通常情况下，一个 id 字段只需要被索引为一个 keyword，而一个 body 字段只需要被索引为一个 text 字段。可以通过在字符串字段上配置显式映射或设置将字符串字段映射为文本\(text\)或关键字\(keyword\)的动态模板来禁用此功能。例如下面的模板，可以用来将 strings 字段映射为关键字：

PUT ${INDEX\_NAME}

{

"mappings": {

```
"type": {

  "dynamic\_templates": \[

    {

      "strings": {

        "match\_mapping\_type": "string",

        "mapping": {

          "type": "keyword"

        }

      }

    }

  \]

}
```

}

}

1. 禁用 \_all 字段

\_all 字段是由所有字段拼接成的超级字段，如果在查询中已知需要查询的字段，就可以考虑禁用它。



PUT /\_template/${TEMPLATE\_NAME}

{

"template": "${TEMPLATE\_PATTERN}",

"settings" : {...},

"mappings": {

```
"type\_1": {

  "\_all": {

     "enabled": false

   },

  "properties": {...}
```

}

},

"version"  : 1

}

1. 使用 best\_compression

\_source 字段和 stored fields 会占用大量存储，可以考虑使用 best\_compression 进行压缩。默认的压缩方式为 LZ4，但需要更高压缩比的话，可以通过 inex.codec 进行设置，修改为 DEFLATE，在 force merge 后生效：



\# Step1. 修改压缩算法为 best\_compression

PUT  /\_template/${TEMPLATE\_NAME}

{

"template":"${TEMPLATE\_PATTERN}",

"settings" : {

```
"index.codec" : "best\_compression"
```

},

"version"  : 1

}

\# Step2. force merge

POST /${INDEX\_NAME}/\_forcemerge?max\_num\_segments=1&wait\_for\_completion=true

POST /${INDEX\_PATTERN}/\_forcemerge?max\_num\_segments=1&wait\_for\_completion=true

1. 使用最优数据格式

我们为数字数据选择的类型可能会对磁盘使用量产生重大影响。 首先，应使用整数类型（byte，short，integer或long）来存储整数，浮点数应该存储在 scaled\_float 中，或者存储在适合用例的最小类型中：使用 float 而不是 double，使用 half\_float 而不是 float。



PUT /\_template/${TEMPLATE\_NAME}

{

"template": "${TEMPLATE\_PATTERN}",

"settings" : {...},

"mappings": {

```
"type\_1": {

  "${FIELD\_NAME}": {

     "type": "integer"

   },

  "properties": {...}
```

}

},

"version"  : 1

}

