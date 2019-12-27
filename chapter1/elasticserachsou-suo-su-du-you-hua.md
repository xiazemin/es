**Elasticserach搜索速度优化**

**1. 避免Join和Parent-Child**  
Join会使查询慢数倍、 Parent-Child会使查询慢数百倍，请在进行 query 语句编写的时候尽量避免。

**2. 映射**  
某些数据本身是数字，但并不意味着它应该总是被映射为一个数字字段。 通常存储着标识符的字段（如ISBN）或来自另一个数据库的数字型记录，可能映射为 keyword 而不是 integer 或者 long 会更好些。

**3. 避免使用 Scripts**  
之前 Groovy 脚本曝出了很大的漏洞，总的来说是需要避免使用的。如果必须要使用，尽量用 5.X 以上版本自带的 painless 和 expressions 引擎。

**4. 根据四舍五入的日期进行查询**  
根据 timestamp 字段进行的查询通常不可缓存，因为匹配的范围始终在变化。 但就用户体验而言，以四舍五入对日期进行转换通常是可接受的，这样可以有效利用系统缓存。举例说明，有以下查询：

| 1234567891011121314151617181920 | `PUT index/type/1{"my_date":"2016-05-11T16:30:55.328Z"}GET index/_search{"query": {"constant_score": {"filter": {"range": {"my_date": {"gte":"now-1h","lte":"now"}}}}}}` |
| :--- | :--- |


可以对时间范围进行替换：

| 123456789101112131415 | `GET index/_search{"query": {"constant_score": {"filter": {"range": {"my_date": {"gte":"now-1h/m","lte":"now/m"}}}}}}` |
| :--- | :--- |


在这种情况下，我们四舍五入到分钟，所以如果当前时间是 16:31:29 ，范围查询将匹配 my\_date 字段的值在 15:31:00 和16:31:59 之间的所有内容。 如果多个用户在同一分钟内运行包含这个范围的查询，查询缓存可以帮助加快速度。 用于四舍五入的时间间隔越长，查询缓存可以提供的帮助就越多，但要注意过于积极的舍入也可能会伤害用户体验。

为了能够利用查询缓存，建议将范围分割成大的可缓存部分和更小的不可缓存的部分，如下所示：

| 12345678910111213141516171819202122232425262728293031323334353637 | `GET index/_search{"query": {"constant_score": {"filter": {"bool": {"should": [{"range": {"my_date": {"gte":"now-1h","lte":"now-1h/m"}}},{"range": {"my_date": {"gt":"now-1h/m","lt":"now/m"}}},{"range": {"my_date": {"gte":"now/m","lte":"now"}}}]}}}}}` |
| :--- | :--- |


然而，这种做法可能会使查询在某些情况下运行速度较慢，因为由 bool 查询引入的开销可能会因更好地利用查询缓存而失败。

**5. 对只读 indices 进行 force merge**  
建议将只读索引被合并到一个单独的分段中。 基于时间的索引通常就是这种情况：只有当前时间索引会写入数据，而旧索引是只读索引。

**6. 预热 global ordinals**  
全局序号\(global ordinals\)是用于在关键字\(keyword\)字段上运行 terms aggregations 的数据结构。 由于 ElasticSearch 不知道聚合使用哪些字段、哪些字段不使用，所以它们在内存中被加载得很慢。 我们可以通过下面的 API 来告诉 ElasticSearch 通过配置映射来在 refresh 的时候加载全局序号：

| 12345678910111213 | `PUT index{"mappings": {"type": {"properties": {"foo": {"type":"keyword","eager_global_ordinals":true}}}}}` |
| :--- | :--- |




