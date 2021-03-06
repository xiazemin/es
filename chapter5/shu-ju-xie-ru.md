**数据写入**

**写入过程**

几个概念：

* 内存buffer

* translog

* 文件系统缓冲区

* refresh
* segment（段）
* commit
* flush

![](https://pic2.zhimg.com/80/v2-189575562c84f184e964336925492f95_hd.jpg)

translog

写入ES的数据首先会被写入translog文件，该文件持久化到磁盘，保证服务器宕机的时候数据不会丢失，由于顺序写磁盘，速度也会很快。

* 同步写入：每次写入请求执行的时候，translog在fsync到磁盘之后，才会给客户端返回成功
* 异步写入：写入请求缓存在内存中，每经过固定时间之后才会fsync到磁盘，写入量很大，对于数据的完整性要求又不是非常严格的情况下，可以开启异步写入

refresh

经过固定的时间，或者手动触发之后，将内存中的数据构建索引生成segment，写入文件系统缓冲区

commit/flush

超过固定的时间，或者translog文件过大之后，触发flush操作：

* 内存的buffer被清空，相当于进行一次refresh
* 文件系统缓冲区中所有segment刷写到磁盘
* 将一个包含所有段列表的新的提交点写入磁盘
* 启动或重新打开一个索引的过程中使用这个提交点来判断哪些segment隶属于当前分片
* 删除旧的translog，开启新的translog

merge

上面提到，每次refresh的时候，都会在文件系统缓冲区中生成一个segment，后续flush触发的时候持久化到磁盘。所以，随着数据的写入，尤其是refresh的时间设置的很短的时候，磁盘中会生成越来越多的segment：

* segment数目太多会带来较大的麻烦。 每一个segment都会消耗文件句柄、内存和cpu运行周期。
* 更重要的是，每个搜索请求都必须轮流检查每个segment，所以segment越多，搜索也就越慢。

merge的过程大致描述如下：

* 磁盘上两个小segment：A和B，内存中又生成了一个小segment：C
* A,B被读取到内存中，与内存中的C进行merge，生成了新的更大的segment：D
* 触发commit操作，D被fsync到磁盘
* 创建新的提交点，删除A和B，新增D
* 删除磁盘中的A和B

**删改操作**

segment的不可变性的好处

* segment的读写不需要加锁
* 常驻文件系统缓存（堆外内存）
* 查询的filter缓存可以常驻内存（堆内存）

删除

磁盘上的每个segment都有一个.del文件与它相关联。当发送删除请求时，该文档未被真正删除，而是在.del文件中标记为已删除。此文档可能仍然能被搜索到，但会从结果中过滤掉。当segment合并时，在.del文件中标记为已删除的文档不会被包括在新的segment中，也就是说merge的时候会真正删除被删除的文档。

更新

创建新文档时，Elasticsearch将为该文档分配一个版本号。对文档的每次更改都会产生一个新的版本号。当执行更新时，旧版本在.del文件中被标记为已删除，并且新版本在新的segment中写入索引。旧版本可能仍然与搜索查询匹配，但是从结果中将其过滤掉。

**版本控制**

通过添加版本号的乐观锁机制保证高并发的时候，数据更新不会出现线程安全的问题，避免数据更新被覆盖之类的异常出现。

使用内部版本号：删除或者更新数据的时候，携带\_version参数，如果文档的最新版本不是这个版本号，那么操作会失败，这个版本号是ES内部自动生成的，每次操作之后都会递增一。

PUT /website/blog/1?version=1

{

"title": "My first blog entry",

"text":  "Starting to get the hang of this..."

}

使用外部版本号：ES默认采用递增的整数作为版本号，也可以通过外部自定义整数（long类型）作为版本号，例如时间戳。通过添加参数version\_type=external，可以使用自定义版本号。内部版本号使用的时候，更新或者删除操作需要携带ES索引当前最新的版本号，匹配上了才能成功操作。但是外部版本号使用的时候，可以将版本号更新为指定的值。

PUT /website/blog/2?version=5&version\_type=external

{

"title": "My first external blog entry",

"text":  "Starting to get the hang of this..."

}

原始文档存储（行式存储）

  


![](https://pic4.zhimg.com/80/v2-6cda4d6eb078571fe5d7c56e7c6dd107_hd.jpg)

**fdt文件**

文档内容的物理存储文件，由多个chunk组成，Lucene索引文档时，先缓存文档，缓存大于16KB时，就会把文档压缩存储。  


![](https://pic3.zhimg.com/80/v2-c9fb74c7a847f4704f7f9b2554c97c22_hd.jpg)

**fdx文件**

文档内容的位置索引，由多个block组成：

* 1024个chunk归为一个block
* block记录chunk的起始文档ID，以及chunk在fdt中的位置
 

![](https://pic4.zhimg.com/80/v2-fc6eca00a128837cd5288fb8f5a56303_hd.jpg)

**fnm文件**

文档元数据信息，包括文档字段的名称、类型、数量等。

**原始文档的查询**

![](https://pic1.zhimg.com/80/v2-aeda0b36ccd362d48647d05086710c38_hd.jpg)

注意问题：lucene对原始文件的存放是行式存储，并且为了提高空间利用率，是多文档一起压缩，因此取文档时需要读入和解压额外文档，因此取文档过程非常依赖CPU以及随机IO。

**相关设置**

压缩方式的设置

原始文档的存储对应\_source字段，是默认开启的，会占用大量的磁盘空间，上面提到的chunk中的文档压缩，ES默认采用的是LZ4，如果想要提高压缩率，可以将设置改成best\_compression。

```
index.codec: best_compression
```

特定字段的内容存储

查询的时候，如果想要获取原始字段，需要在\_source中获取，因为所有的字段存储在一起，所以获取完整的文档内容与获取其中某个字段，在资源消耗上几乎相同，只是返回给客户端的时候，减少了一定量的网络IO。

ES提供了特定字段内容存储的设置，在设置mappings的时候可以开启，默认是false。如果你的文档内容很大，而其中某个字段的内容有需要经常获取，可以设置开启，将该字段的内容单独存储

PUT my\_index

{

  "mappings": {

    "\_doc": {

      "properties": {

        "title": {

          "type": "text",

          "store": true 

        }

      }

    }

  }

}

