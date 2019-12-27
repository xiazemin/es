**Elasticsearch常用插件**

**elasticsearch-head 插件**  
一个elasticsearch的集群管理工具，它是完全由html5编写的独立网页程序，你可以通过插件把它集成到es。  
项目地址：[https://github.com/mobz/elasticsearch-head](https://github.com/mobz/elasticsearch-head)

插件安装方法1

| 123 | `elasticsearch/bin/plugininstallmobz/elasticsearch-head重启elasticsearch打开http://localhost:9200/_plugin/head/` |
| :--- | :--- |


插件安装方法2

| 12345 | `根据地址https://github.com/mobz/elasticsearch-head下载zip解压建立elasticsearch/plugins/head/_site文件将解压后的elasticsearch-head-master文件夹下的文件copy到_site重启elasticsearch打开http://localhost:9200/_plugin/head/` |
| :--- | :--- |


**bigdesk插件**  
elasticsearch的一个集群监控工具，可以通过它来查看es集群的各种状态，如：cpu、内存使用情况，索引数据、搜索情况，http连接数等。  
项目地址：[https://github.com/hlstudio/bigdesk](https://github.com/hlstudio/bigdesk)

插件安装方法1

| 123 | `elasticsearch/bin/plugininstallhlstudio/bigdesk重启elasticsearch打开http://localhost:9200/_plugin/bigdesk/` |
| :--- | :--- |


插件安装方法2

| 12345 | `https://github.com/hlstudio/bigdesk下载zip 解压建立elasticsearch-1.0.0\plugins\bigdesk\_site文件将解压后的bigdesk-master文件夹下的文件copy到_site重启elasticsearch打开http://localhost:9200/_plugin/bigdesk/` |
| :--- | :--- |


**Kopf 插件**  
一个ElasticSearch的管理工具，它也提供了对ES集群操作的API。  
项目地址：[https://github.com/lmenezes/elasticsearch-kopf](https://github.com/lmenezes/elasticsearch-kopf)

插件安装方法

| 123 | `elasticsearch/bin/plugininstalllmenezes/elasticsearch-kopf重启elasticsearch打开http://localhost:9200/_plugin/kopf/` |
| :--- | :--- |




