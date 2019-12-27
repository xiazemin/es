**ElasticSearch配置**

**1.**数据目录和日志目录，生产环境下应与软件分离

| 12345678 | `#注意：数据目录可以有多个，可以通过逗号分隔指定多个目录。一个索引数据只会放入一个目录中！！path.data:/path/to/data1,/path/to/data2# Path to log files:path.logs:/path/to/logs# Path to where plugins are installed:path.plugins:/path/to/plugins` |
| :--- | :--- |


**2.**所属的集群名，默认为 elasticsearch ，可自定义（最好给生产环境的ES集群改个名字，改名字的目的其实就是防止某台服务器加入了集群这种意外）

| 1 | `cluster.name: kevin_elasticsearch　` |
| :--- | :--- |


**3.**节点名，默认为 UUID前7个字符，可自定义

| 1 | `node.name: kevin_elasticsearch_node01` |
| :--- | :--- |


**4.**network.host  IP绑定，默认绑定的是\["127.0.0.1", "\[::1\]"\]回环地址，集群下要服务间通信，需绑定一个ipv4或ipv6地址或0.0.0.0

| 1 | `network.host: 172.16.60.11` |
| :--- | :--- |


**5.**http.port: 9200-9300  
对外服务的http 端口， 默认 9200-9300 。可以为它指定一个值或一个区间，当为区间时会取用区间第一个可用的端口。

**6.**transport.tcp.port: 9300-9400  
节点间交互的端口， 默认 9300-9400 。可以为它指定一个值或一个区间，当为区间时会取用区间第一个可用的端口。

**7.**Discovery Config 节点发现配置  
ES中默认采用的节点发现方式是 zen（基于组播（多播）、单播）。在应用于生产前有两个重要参数需配置

**8.**discovery.zen.ping.unicast.hosts: \["host1","host2:port","host3\[portX-portY\]"\]  
单播模式下，设置具有master资格的节点列表，新加入的节点向这个列表中的节点发送请求来加入集群。

**9.**discovery.zen.minimum\_master\_nodes: 1  
这个参数控制的是，一个节点需要看到具有master资格的节点的最小数量，然后才能在集群中做操作。官方的推荐值是\(N/2\)+1，其中N是具有master资格的节点的数量。

**10.**transport.tcp.compress: false  
是否压缩tcp传输的数据，默认false

**11.**http.cors.enabled: true  
是否使用http协议对外提供服务，默认true

**12.**http.max\_content\_length: 100mb  
http传输内容的最大容量，默认100mb

**13.**node.master: true  
指定该节点是否可以作为master节点，默认是true。ES集群默认是以第一个节点为master，如果该节点出故障就会重新选举master。

**14.**node.data: true  
该节点是否存索引数据，默认true。

**15.**discover.zen.ping.timeout: 3s  
设置集群中自动发现其他节点时ping连接超时时长，默认为3秒。在网络环境较差的情况下，增加这个值，会增加节点等待响应的时间，从一定程度上会减少误判。

**16.**discovery.zen.ping.multicast.enabled: false  
是否启用多播来发现节点。

**17.**Jvm heap 大小设置  
生产环境中一定要在jvm.options中调大它的jvm内存。

**18.**JVM heap dump path 设置  
生产环境中指定当发生OOM异常时，heap的dump path，好分析问题。在jvm.options中配置：  
-XX:HeapDumpPath=/var/lib/elasticsearch

**五、ElasticSearch安装配置手册**

**1）环境要求**  
Elasticsearch 6.0版本至少需要JDK版本1.8。  
可以使用java -version 命令查看JDK版本。

**2）安装步骤**  
在ES 官网下载ES安装包，现以6.1.1版本为例。  
**1.  **搜索历史版本，找到ES6.1.1版本下载连接。  
[https://www.elastic.co/cn/downloads/past-releases\#elasticsearch](https://www.elastic.co/cn/downloads/past-releases#elasticsearch)

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724134529336-1608798145.png)

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724134535948-21920447.png)

**2.  **将下载好的安装包上传至需要安装的服务器目录下并解压。  
\# cd /opt/gov  
\# tar -zxvf elasticsearch-6.1.1.tar.gz

**3.**  解压完毕  
文件夹elasticsearch-6.1.1 为elasticsearch-6.1.1所在目录。

**3）配置Elasticsearch**  
**1.**  进入Elasticsearch-6.1.1所在目录结构，其中config文件夹为es配置文件所在位置，进入config文件夹。

**2.**  修改elasticsearch.yml文件中的集群名称和节点名称

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724140407895-1025647154.png)

cluster.name（集群名称） 和node.name（节点名称） 可以自己配置简单易懂的值。node.name为节点名称，如果多个es实例,可加上数字1.2.3进行区分，方便查看日志和区分节点，简单易懂。如果该es集群用于落地skywalking的apm-collector数据，建议将cluster.name配置为cluster.name: CollectorDBCluster，即该名称需要和skywalking的collector配置文件一致。同一集群下多个es节点的cluster.name应该保持一致。

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724140449992-110975529.png)

node.master指定该节点是否有资格被选举成为master，默认是true，elasticsearch默认集群中的第一台启动的机器为master，如果这台机挂了就会重新选举master。  
node.data指定该节点是否存储索引数据，默认为true。如果节点配置node.master:false并且node.data: false，则该节点将起到负载均衡的作用。

**3.**  修改elasticsearch.yml文件中的数据和日志保存目录

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724140616167-840286919.png)

此处作为实例将data目录和logs目录建立在了elasticsearch-6.1.1下，这样是危险的。当es被卸载，数据和日志将完全丢失。可以根据具体环境将数据目录和日志目录保存到其他路径下以确保数据和日志完整、安全。

**4.**  修改elasticsearch.yml文件中的network信息

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724140719735-955483941.png)

如果一台主机将要安装多个es实例，请自行更改端口，以免端口被占用。http.port默认端口为9200。transport.tcp.port 默认端口为9300。

**5.**  修改elasticsearch.yml文件中的discovery信息

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724140809475-1454245628.png)

discovery.zen.ping.unicast.hosts: \["192.168.0.8:9300", "192.168.0.9:9300","192.168.0.10:9300"\]  
如果多个es实例组成集群，各节点ip+port信息用逗号分隔。其中port为各es实例中配置的transport.tcp.port。

discovery.zen.minimum\_master\_nodes: 1  
设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）

**6.**  在elasticsearch.yml文件末尾添加配置

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724163508971-1800030465.png)

bootstrap.memory\_lock: false  
是否锁住内存。因为当jvm开始swapping时es的效率会降低，配置为true时要允许elasticsearch的进程可以锁住内存，同时一定要保证机器有足够的内存分配给es。如果不熟悉机器所处环境，建议配置为false。

bootstrap.system\_call\_filter: false  
Centos6不支持SecComp，而ES5.2.0版本默认bootstrap.system\_call\_filter为true  
禁用：在elasticsearch.yml中配置bootstrap.system\_call\_filter为false，注意要在Memory的后面配置该选项。

http.cors.enabled: true  
是否支持跨域，默认为false。

http.cors.allow-origin: "\*"  
当设置允许跨域，默认为\*,表示支持所有域名，如果我们只是允许某些网站能访问，那么可以使用正则表达式。比如只允许本地地址 /https?:\/\/localhost\(:\[0-9\]+\)?/

**六、启动Elasticsearch**

**1.**进入elasticsearch-6.1.1下的bin目录。  
**2.**运行 ./elasticsearch -d 命令。  
**3.**如果出现./elasticsearch: Permission denied ，这时执行没有权限。  
需要授权执行命令:chmod +x bin/elasticsearch 。  
**4.**再次执行./elasticsearch -d即可启动  
**5.**在我们配置的es日志目录中，查看日志文件elasticsearch.log，确保es启动成功。

![](https://img2018.cnblogs.com/blog/907596/201907/907596-20190724171743169-1912819086.png)

**6.**查看elasticssearch进程,   运行"ps -aux\|grep elasticsearch" 命令即可

