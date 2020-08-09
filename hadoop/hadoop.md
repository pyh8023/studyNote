# 1. 大数据初识

> 学习知识的时候要去搞明白它存在的意义，这样学习成本才会低！

Apache Hadoop项目为可靠的、可扩展的分布式计算开发开源软件。

## 1.1 大数据关心的重点

- 分而治之
- 并行计算
- 计算向数据移动
- 数据本地读取

## 1.2 hadoop时间简史

 《The Google File System 》 2003年 

 《MapReduce: Simplified Data Processing on Large Clusters》 2004年 

 《Bigtable: A Distributed Storage System for Structured Data》 2006年

   Hadoop由 Apache Software Foundation 于 2005 年秋天作为Lucene的子项目Nutch的一部分正式引入。

   2006 年 3 月份，Map/Reduce 和 Nutch Distributed File System (NDFS) 分别被纳入称为 Hadoop 的项目中。

  Cloudera公司在2008年开始提供基于Hadoop的软件和服务。2016年10月hadoop-2.6.5

  2017年12月hadoop-3.0.0

hadoop.apache.org

## 1.3 Hadoop项目/生态

![image-20200809115125141](./images/image-20200809115125141.png)

hadoop包括4个模块：

1. Hadoop Common

2. Hadoop Distributed File System (HDFS)
3. Hadoop YARN
4. Hadoop MapReduce

Apache中其他与hadoop相关的项目包括:

1. **Ambar** : 一个基于web的用于提供、管理和监视Apache Hadoop集群的工具，包括对Hadoop HDFS、Hadoop MapReduce、Hive、HCatalog、HBase、ZooKeeper、Oozie、Pig和Sqoop的支持。Ambari还提供了一个查看集群运行状况(如heatmaps)的仪表板，以及可视化查看MapReduce、Pig和Hive应用程序的能力，以及以用户友好的方式诊断其性能特征的功能。

2. **Avro**: 数据序列化系统

3. **Cassandra** : 没有单点故障的可伸缩多主数据库。

4. **Chukwa** : 用于管理大型分布式系统的数据收集系统。

5. **HBase** : 可伸缩的分布式数据库，支持大型表的结构化数据存储。

6. **Hive** : 提供数据汇总和特别查询的数据仓库基础设施。

7. **Mahout** : 一个可扩展的机器学习和数据挖掘库。

8. **Pig** : 用于并行计算的高级数据流语言和执行框架。

9. **Spark** : 一个快速和通用的Hadoop数据计算引擎。Spark提供了一个简单而富有表现力的编程模型，它支持广泛的应用程序，包括ETL、机器学习、流处理和图形计算。

10. **Tez** : 一个基于Hadoop YARN的通用数据流编程框架，它提供了一个强大而灵活的引擎，可以执行任意的DAG任务来处理批处理和交互式用例的数据。Hadoop生态系统中的Hive、Pig等框架以及其他商业软件(如ETL工具)正在采用Tez，以取代Hadoop的MapReduce作为底层执行引擎。

11. **ZooKeeper** : 分布式应用程序的高性能协调服务。

    

# 2.Hadoop Distributed File System (HDFS)

## 2.1 存储模型

文件线性按字节切割成块(block)，具有offset，id

文件与文件的block大小可以不一样

一个文件除最后一个block，其他block大小一致

block的大小依据硬件的I/O特性调整

block被分散存放在集群的节点中，具有location

Block具有副本(replication)，没有主从概念，副本不能出现在同一个节点

副本是满足可靠性和性能的关键

文件上传可以指定block大小和副本数，上传后只能修改副本数

一次写入多次读取，不支持修改

支持追加数据

## 2.2 架构设计

HDFS是一个主从(Master/Slaves)架构

由一个NameNode和一些DataNode组成

面向文件包含：文件数据(data)和文件元数据(metadata)

NameNode负责存储和管理文件元数据，并维护了一个层次型的文件目录树

DataNode负责存储文件数据(block块)，并提供block的读写

DataNode与NameNode维持心跳，并汇报自己持有的block信息

Client和NameNode交互文件元数据和DataNode交互文件block数据

## 2.3 角色功能

**NameNode**

- 完全基于内存存储文件元数据、目录结构、文件block的映射
- 需要持久化方案保证数据可靠性
- 提供副本放置策略

**DataNode**

- 基于本地磁盘存储block(文件的形式)
- 并保存block的校验和数据保证block的可靠性
- 与NameNode保持心跳，汇报block列表状态

## 2.4 元数据持久化

任何对文件系统元数据产生修改的操作，Namenode都会使用一种称为EditLog的事务日志记录下来

使用FsImage存储内存所有的元数据状态

使用本地磁盘保存EditLog和FsImage

EditLog具有完整性，数据丢失少，但恢复速度慢，并有体积膨胀风险

FsImage具有恢复速度快，体积与内存数据相当，但不能实时保存，数据丢失多

NameNode使用了FsImage+EditLog整合的方案：

- 滚动将增量的EditLog更新到FsImage，以保证更近时点的FsImage和更小的EditLog体积

## 2.5 安全模式

1. HDFS搭建时会格式化，格式化操作会产生一个空的FsImage

2. 当Namenode启动时，它从硬盘中读取Editlog和FsImage，将所有Editlog中的事务作用在内存中的FsImage上，并将这个新版本的FsImage从内存中保存到本地磁盘上，然后删除旧的Editlog，因为这个旧的Editlog的事务都已经作用在FsImage上了

3. Namenode启动后会进入一个称为安全模式的特殊状态。

4. 处于安全模式的Namenode是不会进行数据块的复制的。

5. Namenode从所有的 Datanode接收心跳信号和块状态报告。

6. 每当Namenode检测确认某个数据块的副本数目达到这个最小值，那么该数据块就会被认为是副本安全(safely replicated)的。

7. 在一定百分比（这个参数可配置）的数据块被Namenode检测确认是安全之后（加上一个额外的30秒等待时间），Namenode将退出安全模式状态。

8. 接下来它会确定还有哪些数据块的副本没有达到指定数目，并将这些数据块复制到其他Datanode上。

**SecondaryNameNode（SNN）**

1. 在非Ha模式下，SNN一般是独立的节点，周期完成对NN的EditLog向FsImage合并，减少EditLog大小，减少NN启动时间

2. 根据配置文件设置的时间间隔fs.checkpoint.period 默认3600秒

3. 根据配置文件设置edits log大小 fs.checkpoint.size 规定edits文件的最大值默认是64MB

![image-20200809152458525](./images/image-20200809152458525.png)

## 2.6 副本放置策略

第一个副本：放置在上传文件的DN；如果是集群外提交，则随机挑选一台磁盘不太满，CPU不太忙的节点。

第二个副本：放置在于第一个副本不同的 机架的节点上。

第三个副本：与第二个副本相同机架的节点。

更多副本：随机节点。

## 2.7 读写流程

**HDFS写流程**

![HDFS写流程](./images/image-20200809152752650.png)

1. Client和NN连接创建文件元数据

2. NN判定元数据是否有效: 比如文件是否存在，是有具有写权限
3. NN处发副本放置策略，返回一个有序的DN列表
4. Client和DN建立Pipeline连接
5. Client将块切分成packet（64KB），并使用chunk（512B）+chucksum校验和（4B）填充
6. Client将packet放入发送队列dataqueue中，并向第一个DN发送
7. 第一个DN收到packet后本地保存并发送给第二个DN
8. 第二个DN收到packet后本地保存并发送给第三个DN
9. 这一个过程中，上游节点同时发送下一个packet
10. 生活中类比工厂的流水线：结论：流式其实也是变种的并行计算
11. Hdfs使用这种传输方式，副本数对于client是透明的
12. 当block传输完成，DN们各自向NN汇报，同时client继续传输下一个block
13. 所以，client的传输和block的汇报也是并行的

**HDFS读流程**

![HDFS读流程](./images/image-20200809153736323.png)

1. 为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本。

   如果在读取程序的同一个机架上有一个副本，那么就读取该副本。

   如果一个HDFS集群跨越多个数据中心，那么客户端也将首先读本地数据中心的副本。

2. 语义：下载一个文件：

   Client和NN交互文件元数据获取fileBlockLocation

   NN会按距离策略排序返回

   Client尝试下载block并校验数据完整性

3. 语义：下载一个文件其实是获取文件的所有的block元数据，那么子集获取某些block应该成立

   Hdfs支持client给出文件的offset自定义连接哪些block的DN，自定义获取数据

   这个是支持计算层的分治、并行计算的核心

## 2.8 安全策略