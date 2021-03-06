## 动态索引

下一个需要解决的问题是如何在保持不可变好处的同时更新倒排索引。答案是，**使用多个索引**。

不是重写整个倒排索引，而是增加额外的索引反映最近的变化。每个倒排索引都可以按顺序查询，从最老的开始，最后把结果聚合。

Elasticsearch底层依赖的Lucene，引入了`per-segment search`的概念。一个段\(`segment`\)是有完整功能的倒排索引，但是现在Lucene中的索引指的是段的集合，再加上提交点\(`commit point`，包括所有段的文件\)，如图1所示。新的文档，在被写入磁盘的段之前，首先写入内存区的索引缓存，如图2、图3所示。

图1：一个提交点和三个索引的Lucene



![](https://user-gold-cdn.xitu.io/2018/8/23/16564d9c6d3028e0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```
索引vs分片
为了避免混淆，需要说明，Lucene索引是Elasticsearch中的分片，Elasticsearch中的索引是分片的集合。
当Elasticsearch搜索索引时，它发送查询请求给该索引下的所有分片，然后过滤这些结果，聚合成全局的结果。

复制代码
```

一个`per-segment search`如下工作:

```
1.新的文档首先写入内存区的索引缓存。
2.不时，这些buffer被提交：
    一个新的段——额外的倒排索引——写入磁盘。
    新的提交点写入磁盘，包括新段的名称。
    磁盘是fsync(文件同步)——所有写操作等待文件系统缓存同步到磁盘，确保它们可以被物理写入。
3.新段被打开，它包含的文档可以被检索
4.内存的缓存被清除，等待接受新的文档。

复制代码
```

图2：内存缓存区有即将提交文档的Lucene索引



![](https://user-gold-cdn.xitu.io/2018/8/23/16564db22efdd194?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



图3：提交后，新的段加到了提交点，缓存被清空



![](https://user-gold-cdn.xitu.io/2018/8/23/16564db774b071db?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



当一个请求被接受，所有段依次查询。所有段上的Term统计信息被聚合，确保每个term和文档的相关性被正确计算。通过这种方式，新的文档以较小的代价加入索引。



