**1. 开启慢查询日志**  
不论是数据库还是搜索引擎，对于问题的排查，开启慢查询日志是十分必要的，ElasticSearch 开启慢查询的方式有多种，但是最常用的是调用模板 API 进行全局设置：

| 1234567891011121314151617181920212223 | `PUT /_template/{TEMPLATE_NAME}{"template":"{INDEX_PATTERN}","settings": {"index.indexing.slowlog.level":"INFO","index.indexing.slowlog.threshold.index.warn":"10s","index.indexing.slowlog.threshold.index.info":"5s","index.indexing.slowlog.threshold.index.debug":"2s","index.indexing.slowlog.threshold.index.trace":"500ms","index.indexing.slowlog.source":"1000","index.search.slowlog.level":"INFO","index.search.slowlog.threshold.query.warn":"10s","index.search.slowlog.threshold.query.info":"5s","index.search.slowlog.threshold.query.debug":"2s","index.search.slowlog.threshold.query.trace":"500ms","index.search.slowlog.threshold.fetch.warn":"1s","index.search.slowlog.threshold.fetch.info":"800ms","index.search.slowlog.threshold.fetch.debug":"500ms","index.search.slowlog.threshold.fetch.trace":"200ms"},"version": 1}` |
| :--- | :--- |


对于已经存在的 index 使用 settings API：

| 123456789101112131415161718 | `PUT {INDEX_PAATERN}/_settings{"index.indexing.slowlog.level":"INFO","index.indexing.slowlog.threshold.index.warn":"10s","index.indexing.slowlog.threshold.index.info":"5s","index.indexing.slowlog.threshold.index.debug":"2s","index.indexing.slowlog.threshold.index.trace":"500ms","index.indexing.slowlog.source":"1000","index.search.slowlog.level":"INFO","index.search.slowlog.threshold.query.warn":"10s","index.search.slowlog.threshold.query.info":"5s","index.search.slowlog.threshold.query.debug":"2s","index.search.slowlog.threshold.query.trace":"500ms","index.search.slowlog.threshold.fetch.warn":"1s","index.search.slowlog.threshold.fetch.info":"800ms","index.search.slowlog.threshold.fetch.debug":"500ms","index.search.slowlog.threshold.fetch.trace":"200ms"}` |
| :--- | :--- |


这样，在日志目录下的慢查询日志就会有输出记录必要的信息了。

