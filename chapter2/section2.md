**Elasticsearch集群健康状态**

Elasticsearch默认在 9200 端口运行，请求 curl http://localhost:9200/ 或者浏览器输入 http://localhost:9200，得到一个 JSON 对象，其中包含当前节点、集群、版本等信息。

| 1234567891011121314151617 | `{"name":"U7fp3O9","cluster_name":"elasticsearch","cluster_uuid":"-Rj8jGQvRIelGd9ckicUOA","version": {"number":"6.8.1","build_flavor":"default","build_type":"zip","build_hash":"1fad4e1","build_date":"2019-06-18T13:16:52.517138Z","build_snapshot":false,"lucene_version":"7.7.0","minimum_wire_compatibility_version":"5.6.0","minimum_index_compatibility_version":"5.0.0"},"tagline":"You Know, for Search"}` |
| :--- | :--- |


要检查群集运行状况，我们可以在 Kibana 控制台中运行以下命令 GET /\_cluster/health，得到如下信息:

| 1234567891011121314151617 | `{"cluster_name":"elasticsearch","status":"yellow","timed_out":false,"number_of_nodes": 1,"number_of_data_nodes": 1,"active_primary_shards": 9,"active_shards": 9,"relocating_shards": 0,"initializing_shards": 0,"unassigned_shards": 5,"delayed_unassigned_shards": 0,"number_of_pending_tasks": 0,"number_of_in_flight_fetch": 0,"task_max_waiting_in_queue_millis": 0,"active_shards_percent_as_number": 64.28571428571429}` |
| :--- | :--- |


集群状态通过 绿，黄，红 来标识：  
**绿色**：集群健康完好，一切功能齐全正常，所有分片和副本都可以正常工作。  
**黄色**：预警状态，所有主分片功能正常，但至少有一个副本是不能正常工作的。此时集群是可以正常工作的，但是高可用性在某种程度上会受影响。  
**红色**：集群不可正常使用。某个或某些分片及其副本异常不可用，这时集群的查询操作还能执行，但是返回的结果会不准确。对于分配到这个分片的写入请求将会报错，最终会导致数据的丢失。

当集群状态为红色时，它将会继续从可用的分片提供搜索请求服务，但是你需要尽快修复那些未分配的分片。

