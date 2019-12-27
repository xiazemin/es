之前描述了 ElasticSearch 在内存管理方面的优化，接下来梳理下如何对写入性能进行优化，写入性能的优化也和 HBase 类似，无非就是增加吞吐，而增加吞吐的方法就是增大刷写间隔、合理设置线程数量、开启异步刷写（允许数据丢失的情况下）。

**1. 增大刷写间隔**  
通过修改主配置文件 elasticsearch.yml 或者 Rest API 都可以对 index.refresh\_interval进行修改，增大该属性可以提升写入吞吐。

| 123456789101112 | `PUT /_template/{TEMPLATE_NAME}{"template":"{INDEX_PATTERN}","settings": {"index.refresh_interval":"30s"}}PUT {INDEX_PAATERN}/_settings{"index.refresh_interval":"30s"}` |
| :--- | :--- |


**2. 合理设置线程数量**  
调整 elasticsearch.yml ，对 bulk/flush 线程池进行调优，根据本机实际配置：

| 1234 | `threadpool.bulk.type:fixedthreadpool.bulk.size:8#(CPU核数)threadpool.flush.type:fixedthreadpool.flush.size:8#(CPU核数)` |
| :--- | :--- |


**3. 开启异步刷写**  
如果允许数据丢失，可以对特定 index 开启异步刷写：

| 123456789101112 | `PUT /_template/{TEMPLATE_NAME}{"template":"{INDEX_PATTERN}","settings": {"index.translog.durability":"async"}}PUT  {INDEX_PAATERN}/_settings{"index.translog.durability":"async"}` |
| :--- | :--- |




