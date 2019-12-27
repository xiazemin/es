**Elasticserach性能优化**

**1.  硬件选择**  
目前公司的物理机机型在CPU和内存方面都满足需求，建议使用SSD机型。原因在于，可以快速把 Lucene 的索引文件加载入内存（这在宕机恢复的情况下尤为明显），减少 IO 负载和 IO wait以便CPU不总是在等待IO中断。建议使用多裸盘而非raid，因为 ElasticSearch 本身就支持多目录，raid 要么牺牲空间要么牺牲可用性。

**2. 系统配置**  
ElasticSearch 理论上必须单独部署，并且会独占几乎所有系统资源，因此需要对系统进行配置，以保证运行 ElasticSearch 的用户可以使用足够多的资源。生产集群需要调整的配置如下：  
**1.**设置 JVM 堆大小；  
**2.**关闭 swap；  
**3.**增加文件描述符；  
**4.**保证足够的虚存；  
**5.**保证足够的线程；  
**6.**暂时不建议使用G1GC；

**3. 设置 JVM 堆大小**  
ElasticSearch 需要有足够的 JVM 堆支撑索引数据的加载，对于公司的机型来说，因为都是大于 128GB 的，所以推荐的配置是 32GB（如果 JVM 以不等的初始和最大堆大小启动，则在系统使用过程中可能会因为 JVM 堆的大小调整而容易中断。 为了避免这些调整大小的暂停，最好使用初始堆大小等于最大堆大小的 JVM 来启动），预留足够的 IO Cache 给 Lucene（官方建议超过一半的内存需要预留）。

**4. 关闭 swap & 禁用交换**  
必须要关闭 swap，因为在物理内存不足时，如果发生 FGC，在回收虚拟内存的时候会造成长时间的 stop-the-world，最严重的后果是造成集群雪崩。公司的默认模板是关闭的，但是要巡检一遍，避免有些机器存在问题。设置方法：

| 12345678910 | `Step1. root 用户临时关闭# swapoff -a# sysctl vm.swappiness=0Step2. 修改/etc/fstab，注释掉 swap 这行Step3. 修改/etc/sysctl.conf，添加：vm.swappiness = 0Step4. 确认是否生效# sysctl vm.swappiness` |
| :--- | :--- |


也可以通过修改 yml 配置文件的方式从 ElasticSearch 层面禁止物理内存和交换区之间交换内存，修改 ${PATH\_TO\_ES\_HOME}/config/elasticsearch.yml，添加：  
bootstrap.memory\_lock: true

==========================小提示=======================  
Linux 把它的物理 RAM 分成多个内存块，称之为分页。内存交换（swapping）是这样一个过程，它把内存分页复制到预先设定的叫做交换区的硬盘空间上，以此释放内存分页。物理内存和交换区加起来的大小就是虚拟内存的可用额度。

内存交换有个缺点，跟内存比起来硬盘非常慢。内存的读写速度以纳秒来计算，而硬盘是以毫秒来计算，所以访问硬盘比访问内存要慢几万倍。交换次数越多，进程就越慢，所以应该不惜一切代价避免内存交换的发生。

ElasticSearch 的 memory\_lock 属性允许 Elasticsearch 节点不交换内存。（注意只有Linux/Unix系统可设置。）这个属性可以在yml文件中设置。  
======================================================

**5. 增加文件描述符**  
单个用户可用的最大进程数量\(软限制\)&单个用户可用的最大进程数量\(硬限制\)，超过软限制会有警告，但是无法超过硬限制。 ElasticSearch 会使用大量的文件句柄，如果超过限制可能会造成宕机或者数据缺失。

文件描述符是用于跟踪打开“文件”的 Unix 结构体。在Unix中，一切都皆文件。 例如，“文件”可以是物理文件，虚拟文件（例如/proc/loadavg）或网络套接字。 ElasticSearch 需要大量的文件描述符（例如，每个 shard 由多个 segment 和其他文件组成，以及到其他节点的 socket 连接等）。

设置方法（假设是 admin 用户启动的 ElasticSearch 进程）：

| 12345678910 | `# Step1. 修改 /etc/security/limits.conf，添加：admin soft nofile 65536admin hard nofile 65536# Step2. 确认是否生效su- adminulimit-n# Step3. 通过 rest 确认是否生效GET/_nodes/stats/process?filter_path=**.max_file_descriptors` |
| :--- | :--- |


**6. 保证足够的虚存**  
单进程最多可以占用的内存区域，默认为 65536。Elasticsearch 默认会使用 mmapfs 去存储 indices，默认的 65536 过少，会造成 OOM 异常。设置方法：

| 12345678 | `# Step1. root 用户修改临时参数sudosysctl -w vm.max_map_count=262144# Step2. 修改 /etc/sysctl.conf，在文末添加：vm.max_map_count = 262144# Step3. 确认是否生效sudosysctl vm.max_map_count` |
| :--- | :--- |


**7. 保证足够的线程**  
Elasticsearch 通过将请求分成几个阶段，并交给不同的线程池执行（Elasticsearch 中有各种不同的线程池执行器）。 因此，Elasticsearch 需要创建大量线程的能力。进程可创建线程的最大数量确保 Elasticsearch 进程有权在正常使用情况下创建足够的线程。 这可以通过`/etc/security/limits.conf` 使用 nproc 设置来完成。设置方法:

| 12 | `修改/etc/security/limits.d/90-nproc.conf，添加：admin soft nproc 2048` |
| :--- | :--- |


**8. 暂时不建议使用G1GC**  
已知 JDK 8 附带的 HotSpot JVM 的早期版本在启用 G1GC 收集器时会导致索引损坏。受影响的版本是早于 JDK 8u40 附带的HotSpot 的版本，出于稳定性的考虑暂时不建议使用。

