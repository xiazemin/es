[CCR 围绕主动-被动索引模型设计](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/ccr-overview.html)。一个 Elasticsearch 集群中的索引可以配置为从另一个 Elasticsearch 集群中的索引复制更改。复制更改的索引称为“追随者索引”，被复制的索引称为“领导者索引”。追随者索引是被动的，它可以服务于读取请求和搜索，但不能接受直接写入，只有领导者索引可随时接受直接写入。由于 CCR 在索引级别进行管理，因此，集群可以包含领导者索引和追随者索引。采用这种方式，您可以按某一方向（例如，从美国集群复制到欧洲集群）复制部分索引，并按另一方向（从欧洲集群复制到美国集群）复制其他索引，以此来解决部分主动-主动用例。

[复制在分片级别完成](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/ccr-overview.html#_the_mechanics_of_replication)；追随者索引中的每个分片将从领导者索引中的相应分片拉取更改，这意味着追随者索引的分片数量与领导者索引相同。追随者会复制所有操作，以便复制创建、更新或删除文档的操作。复制是几乎实时完成的；一旦[分片上的全局检查点前进](https://www.elastic.co/cn/blog/elasticsearch-sequence-ids-6-0)，操作即可被追随分片复制。追随分片可批量高效地拉取操作和创建索引，并且能够并行执行拉取更改的多个请求。这些读取请求可以由主分片及其副本提供服务，除了从分片读取之外，不要对领导者施加额外的负载。此设计能够使 CCR 随生产负载进行扩展，以便您可以持续尽享在 Elasticsearch 中体会到的（和预期的）高吞吐量索引编制速率。

CCR 支持新建的索引和现有索引。最初配置追随者索引时，它会从领导者索引复制底层文件，以便[从领导者索引启动自身](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/remote-recovery.html)，这类似于副本从主数据中恢复的过程。此恢复过程完成后，CCR 将从领导者复制任何其他操作。映射和设置更改将根据需要自动从领导者索引进行复制。

有时 CCR 可能会遇到错误场景（例如网络故障）。CCR 能够自动将这些错误分类为可恢复错误和致命错误。发生可恢复错误时，CCR 即进入重试循环，一旦导致故障的情况得到解决，CCR 将立即继续复制。

复制状态可以[通过专用 API 进行监控](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/ccr-overview.html#_inspecting_the_progress_of_replication)。通过此 API，您可以监控追随者跟踪领导者的紧密程度，查看有关 CCR 性能的详细统计信息，并跟踪需要您注意的任何错误。

我们已将 CCR 与 Kibana 中的监测和管理应用进行了集成。监测 UI 可使您查看 CCR 进度和错误报告。

![](https://images.contentstack.io/v3/assets/bltf7afce26b89a5b33/blt5f4199278fe0732e/5c9d0ca9e71fdc123a206e20/download "Elasticsearch CCR Monitoring UI in Kibana")

Kibana 中的 Elasticsearch CCR 监测 UI

管理 UI 可使您配置远程集群，配置追随者索引，以及管理自动追随者模式，以实现自动复制索引。

![](https://images.contentstack.io/v3/assets/bltf7afce26b89a5b33/blt52c5ee3f73f080a1/5c9d0c19aef178d33948d77b/download "Elasticsearch CCR Management UI in Kibana")

Kibana 中的 Elasticsearch CCR 管理 UI

## 追随最新的索引

很多用户都有需要定期创建新索引的工作负载。例如，Filebeat 推送的日志文件中的每日索引，或[索引生命周期管理](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/index-lifecycle-management.html)自动滚动更新的索引。我们已将[自动追随功能](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/ccr-auto-follow.html)直接构建到 CCR 中，从而无需手动创建追随者索引来从源集群复制这些索引。此功能允许您将索引模式配置为自动从源集群中复制。CCR 将监测源集群中与这些模式匹配的索引，并将追随索引配置为复制这些匹配的领导者索引。

此外，我们还集成了 CCR 和 ILM，以使 CCR 可以复制基于时间的索引，并通过 ILM 在源和目标集群中进行管理。例如，ILM 了解 CCR 何时复制领导者索引，因此，会小心管理破坏性操作（如缩小和删除索引），直到 CCR 完成复制。

## 不了解历史记录的用户

为了使 CCR 能够复制更改，我们需要有领导者索引分片上的操作历史记录，以及每个分片上的指针，以了解可以安全复制的操作。此操作历史记录按序列 ID 控制，指针称为[全局检查点](https://www.elastic.co/cn/blog/elasticsearch-sequence-ids-6-0)。不过这其中有一定的[复杂性](https://www.elastic.co/cn/blog/lucenes-handling-of-deleted-documents)。在 Lucene 中更新或删除一个文档时，Lucene 将做一些标记，以记录该文档已删除。该文档将保留在磁盘上，直到未来的合并操作合并已删除的文档。如果 CCR 在合并删除内容之前复制此操作，则一切顺利。但是，合并发生在其自己的生命周期内，这意味着在 CCR 有机会复制该操作前，可能会合并已删除的文档。如果无法控制何时合并已删除的文档，则 CCR 可能会遗漏操作，且无法完全将操作历史记录复制到追随者索引。在 CCR 设计阶段早期，我们原计划使用[Elasticsearch 事务日志](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/index-modules-translog.html)作为这些操作历史记录的源，这将有助于避开这一问题。但我们很快意识到，事务日志并不适用于 CCR 高效执行所需的访问模式。我们考虑了在事务日志之上及旁边放置额外的数据结构，以实现我们所需的性能，但此方法有一些限制。首先，它可能会增加系统中某个最重要组件的复杂性，这有悖于我们的工程原则。其次，它会束缚我们打算基于操作历史记录构建的[未来更改](https://github.com/elastic/elasticsearch/issues/1242)，这样我们需要强制限制可以对操作历史记录执行的搜索类型，或基于事务日志重新实现所有 Lucene。凭借这种洞察力，我们[意识到需要在 Lucene 中原生构建](https://issues.apache.org/jira/browse/LUCENE-8198)，使我们能够控制何时合并已删除文档的功能，从而高效地将操作历史记录推送到 Lucene 中。我们称这种技术为“软删除”。这项对 Lucene 的投资需要数年才能获得回报，因为不仅 CCR 是基于其构建的，我们还在修改基于软删除的复制模型，而且即将进行的[更改 API](https://github.com/elastic/elasticsearch/issues/1242)也将基于它们实现。[领导者索引需要能够支持软删除](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/ccr-requirements.html#ccr-requirements)。

接下来，当在领导者上合并软删除的文档时，追随者将能够施加影响。为此，我们引入了[分片历史记录保留租约](https://github.com/elastic/elasticsearch/issues/37165)。通过分片历史记录保留租约，追随者可以在其当前所在的历史记录中对领导者的操作历史记录进行标记。领导者分片知道该标记之下的操作可以安全地合并，但在该标记之上的任何操作都必须保留，直到追随者有机会复制它们为止。这些标记可确保在追随者临时脱机的情况下，领导者将保留尚未复制的操作。由于保留此历史记录需要在领导者上有额外存储，因此，这些标记[仅在限定期限内有效](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/ccr-requirements.html#ccr-overview-soft-deletes)，此期限后标记将过期，领导者分片就可自由合并历史记录。您可以根据在追随者脱机时要保留的额外存储大小，以及您希望在追随者必须从领导者重新启动之前接受追随者脱机多长时间来调整这一期限。

