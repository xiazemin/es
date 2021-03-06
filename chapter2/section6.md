# Elasticsearch 中的跨集群复制

Elasticsearch 中的跨集群复制支持 Elasticsearch 和 Elastic Stack 中的各种任务关键型用例：

* **灾难恢复 \(DR\) / 高可用性 \(HA\)**：对于许多任务关键型应用程序，都需要能够承受住数据中心或区域服务中断的影响。以前，此要求在 Elasticsearch 中通过其他技术得到了满足，但这会增加额外的复杂性和管理开销。现在，通过 Elasticsearch 中的原生功能即可满足跨数据中心的 DR/HA 要求，且无需其他技术。

* **数据本地化**：在 Elasticsearch 将数据复制到更靠近用户或应用程序服务器的位置，可以减少延迟，[降低成本](https://developers.google.com/web/fundamentals/performance/why-performance-matters/)。例如，可以将产品目录或参考数据集复制到全球 20 个或更多数据中心，最大限度地缩短数据与应用程序服务器之间的距离。另一个用例是一家在伦敦和纽约都设有办公室的股票交易公司。伦敦办公室的所有交易均在本地写入，并复制到纽约办公室，纽约办公室的所有交易也本地写入，并复制到伦敦办公室。两个办公室都可以全局查看所有交易。

* **集中式报告**
  ：将数据从大量较小型集群复制回集中式报告集群。当跨大型网络进行查询的效率较低时，此功能就可派上用场。例如，一家大型全球银行可能在世界各地拥有 100 个 Elasticsearch 集群，每个集群位于一个不同的银行分支机构内。我们可以使用 CCR 将全球所有 100 个分支银行的事件复制回中心集群，在此对事件进行本地分析和聚合。

在 Elasticsearch 6.7.0 版本之前，这些用例可以部分通过第三方技术来解决，但这种做法很繁琐，会带来大量的管理开销，而且有很大的缺点。通过将跨集群复制原生集成到 Elasticsearch 中，我们让用户摆脱了管理复杂解决方案的负担和缺点，并能提供现有解决方案所不具备的优势（例如，全面错误处理）。我们还提供了 Elasticsearch API 和 Kibana UI 来管理和监测 CCR。

