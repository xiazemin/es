## 删除和更新

段是不可变的，所以文档既不能从旧的段中移除，旧的段也不能更新以反映文档最新的版本。相反，每一个提交点包括一个.del文件，包含了段上已经被删除的文档。

当一个文档被删除，它实际上只是在.del文件中被标记为删除，依然可以匹配查询，但是最终返回之前会被从结果中删除。

文档的更新操作是类似的：当一个文档被更新，旧版本的文档被标记为删除，新版本的文档在新的段中索引。也许该文档的不同版本都会匹配一个查询，但是更老版本会从结果中删除。

## 近实时搜索

因为`per-segment search`机制，索引和搜索一个文档之间是有延迟的。新的文档会在几分钟内可以搜索，但是这依然不够快。

磁盘是瓶颈。提交一个新的段到磁盘需要`fsync`操作，确保段被物理地写入磁盘，即时电源失效也不会丢失数据。但是`fsync`是昂贵的，它不能在每个文档被索引的时就触发。

所以需要一种更轻量级的方式使新的文档可以被搜索，这意味这移除`fsync`。

位于Elasticsearch和磁盘间的是文件系统缓存。如前所说，在内存索引缓存中的文档（图1）被写入新的段（图2），但是新的段首先写入文件系统缓存，这代价很低，之后会被同步到磁盘，这个代价很大。但是一旦一个文件被缓存，它也可以被打开和读取，就像其他文件一样。

图1：内存缓存区有新文档的Lucene索引

![](https://user-gold-cdn.xitu.io/2018/8/23/16564dcd1e5e11c8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Lucene允许新段写入打开，好让它们包括的文档可搜索，而不用执行一次全量提交。这是比提交更轻量的过程，可以经常操作，而不会影响性能。

图2：缓存内容已经写到段中，但是还没提交

![](https://user-gold-cdn.xitu.io/2018/8/23/16564dd33aea3e71?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### refeash API

在Elesticsearch中，这种写入打开一个新段的轻量级过程，叫做refresh。默认情况下，每个分片每秒自动刷新一次。这就是为什么说Elasticsearch是近实时的搜索了：文档的改动不会立即被搜索，但是会在一秒内可见。

这会困扰新用户：他们索引了个文档，尝试搜索它，但是搜不到。解决办法就是执行一次手动刷新，通过API:

POST /\_refresh &lt;1&gt;

POST /blogs/\_refresh &lt;2&gt;

复制代码

&lt;1&gt; refresh所有索引

&lt;2&gt; 只refresh 索引blogs

不是所有的用户都需要每秒刷新一次。也许你使用ES索引百万日志文件，你更想要优化索引的速度，而不是进实时搜索。你可以通过修改配置项refresh\_interval减少刷新的频率：

  PUT /my\_logs

{

  "settings": {

    "refresh\_interval": "30s" &lt;1&gt;

  }

}

复制代码

&lt;1&gt; 每30s refresh一次my\_logs

refresh\_interval可以在存在的索引上动态更新。你在创建大索引的时候可以关闭自动刷新，在要使用索引的时候再打开它。

PUT /my\_logs/\_settings

{ "refresh\_interval": -1 } &lt;1&gt;



PUT /my\_logs/\_settings

{ "refresh\_interval": "1s" } &lt;2&gt;

复制代码

&lt;1&gt; 禁用所有自动refresh

&lt;2&gt; 每秒自动refresh

