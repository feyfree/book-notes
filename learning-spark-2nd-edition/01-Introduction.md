# A Unified Analytics Engine

了解一下 Spark 的来源

## The Genesis of Spark

### 谷歌的大数据和分布式计算

1. 谷歌搜索下诞生的三驾马车: GFS， MapReduce，Bigtable
2. GFS 提供的是容错的和分布式文件系统， BigTable 描述的是在GFS上面可扩展的结构化数据存储，MapReduce 介绍了一种以函数变成基础上的一种新的并行编程模式，处理分布在GFS， BigTable 上面的分布式数据
3. MapReduce 移动代码 (not bringing data to your application): favoring data locality and cluster rack affinity

### Hadoop at Yahoo

GFS 为 HDFS （Hadoop File System）提供了蓝图， HDFS 后来捐给了 Apache 基金会

Apache Hadoop 

1. Hadoop Common
2. MapReduce
3. HDFS
4. Apache Hadoop YARN

Hadoop 的 MapReduce 有一些问题

1. 操作复杂度很高， 很难掌握
2. 它的批处理MapReduce API 冗长，而且需要很多模板化的配置代码， 容错性太差
3. 很多数据任务被拆解为MapReduce tasks， 每对MR pair的中间计算结果为了随后的操作，都会写进磁盘。 这种重复的磁盘IO导致了一些大型的MR job 可能很久 ， 几个小时或者几天

以上导致了Hadoop 的MR 不适合 机器学习， 流计算， 或者是交互式SQL形式的查询

后来工程师们就开发了一些定制的系统 (Apache Hive, Storm, Impala, Giraph, Drill, Mahout 等等)， 增加了更多的特性， 实际上也增加了Hadoop 的操作复杂度， 开发者的学习曲线实际上也更大了。

### Spark‘s Early Years at AMPLab

一开始是论文描述的是在特定任务场景下， Spark 能比 Hadoop MapReduce 快 10-20 倍， 现在肯定快的更多了

Spark 最开始的初心：借鉴Hadoop 的 MapReduce 的思想， 并且增强这个系统

1. 容错性更好， 并行更优雅
2. 支持内存级别， 迭代或者交互式的MapReduce的计算中间结果保存
3. 为多种语言， 以一种编程模型的形式提供更简单， 更兼融的API
4. 一种统一的形式， 支持多种负载

也捐给了Apache， 后来成立了 Databricks



## What is Apache Spark

Apache Spark is a unified engine designed for large-scale distributed data processing, on premises in data centers or in the cloud. 

Spark provides **in-memory storage** for intermediate computations, making it much faster than Hadoop MapReduce. It incorporates libraries with composable APIs for machine learning (MLlib), SQL for interactive queries (Spark SQL), stream process‐ ing (Structured Streaming) for interacting with real-time data, and graph processing (GraphX). Spark’s design philosophy centers around four key characteristics:

* Speed 

* Ease of use 

* Modularity 

* Extensibility

### Speed

1. CPU ， 内存越来越廉价， 多线程， 并行处理越来越发达
2. Spark 用DAG 来构建查询计算，DAG调度器通过构建高效的计算图结构， 使得Job可以拆解成为可以在集群上面并行计算的任务
3. 物理执行引擎Tungsten 
   1. [Project Tungsten: Bringing Apache Spark Closer to Bare Metal](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html)
   2. [TungstenSecret](https://github.com/hustnn/TungstenSecret)

中间计算结果保存在内存，更少的磁盘IO

### Ease Of Use

Spark achieves simplicity by providing a fundamental abstraction of a simple logical data structure called a **Resilient Distributed Dataset** (RDD) upon which all other higher-level structured data abstractions, such as DataFrames and Datasets, are constructed. By providing a set of transformations and actions as operations, Spark offers a simple programming model that you can use to build big data applications in famil‐ iar languages.

### Modularity

Spark operations can be applied across many types of workloads and expressed in any of the supported programming languages: Scala, Java, Python, SQL, and R. Spark offers unified libraries with well-documented APIs that include the following mod‐ ules as core components: Spark SQL, Spark Structured Streaming, Spark MLlib, and GraphX, combining all the workloads running under one engine. We’ll take a closer look at all of these in the next section. 

You can write a single Spark application that can do it all—no need for distinct engines for disparate workloads, no need to learn separate APIs. With Spark, you get a unified processing engine for your workloads

### Extensibility

Spark focuses on its fast, parallel computation engine rather than on storage. Unlike Apache Hadoop, which included both storage and compute, Spark decouples the two. That means you can use Spark to read data stored in myriad sources—Apache Hadoop, Apache Cassandra, Apache HBase, MongoDB, Apache Hive, RDBMSs, and more—and process it all in memory. Spark’s DataFrameReaders and DataFrame Writers can also be extended to read data from other sources, such as Apache Kafka, Kinesis, Azure Storage, and Amazon S3, into its logical data abstraction, on which it can operate. 

The community of Spark developers maintains a list of third-party Spark packages as part of the growing ecosystem (see Figure 1-2). This rich ecosystem of packages includes Spark connectors for a variety of external data sources, performance moni‐ tors, and more.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530093949-spark-connectors.png)

## Unified Analytics

Spark 得了一个 ACM 的奖项，颁奖指出：

Spark replaces all the separate batch processing, graph, stream, and query engines like Storm, Impala, Dremel, Pregel, etc. with a unified stack of components that addresses diverse workloads under a single distributed fast engine.

意思就是 Spark 大一统

### Apache Spark Components as a Unified Stack

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530094827apache-spark-components-as-a-unified-stack.png)

### Apache Spark’s Distributed Execution

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530101732-apache-spark-components-and-architecture.png)

**Spark Driver**

作为负责启动一个SparkSession的一部分， Spark Driver 还有多种其他的角色

1. 负责和cluster manager 的通信
2. 从cluster manager 那边为Spark executors请求资源 (CPU memory)
3. 将 Spark 操作，转化成为 DAG 计算， 并负责调度他们， 分配这些执行任务到Spark executors上面， 当资源分配成功后， Spark Driver 与 这些 executors 直接通信

**Spark Session**

Spark 2.0 中， Spark Session 成为Spark 操作和数据之间统一的 传输导体。 提供了与Spark交互的entry point,  并且简化了与Spark使用。

```scala
// In Scala
import org.apache.spark.sql.SparkSession
// Build SparkSession
val spark = SparkSession
 .builder
 .appName("LearnSpark")
 .config("spark.sql.shuffle.partitions", 6)
 .getOrCreate()
...
// Use the session to read JSON
val people = spark.read.json("...")
...
// Use the session to issue a SQL query
val resultsDF = spark.sql("SELECT city, pop, state, zip FROM table_name")
```

**cluster manager**

1. 内置的 standalone cluster manager
2. apache hadoop yarn
3. apache mesos
4. kubernetes

**spark executor**

1. 运行在cluster 上面的每个工作节点上面
2. 与spark driver 进行通信
3. 负责执行任务
4. 通常情况下， 每个node 只有一个 executor

**deployment modes**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530104657-cheat-sheet-for-spark-deployment-modes.png)

**distributed data and partitions**

数据基本是存储在 HDFS 或者其他的云存储上面

Spark 用一种高层次的逻辑数据抽象 - DateFrame (in memory), 对待每个分片（分区）

尽管这个不是完全实现， Spark executor 更偏向于分配一个能和数据紧密关联 （相近）的一个任务，更好的利用数据的局部性

分片带来更好的并行化， 分布式将数据分解为 chunks 或者 是partitions， 这种允许Spark executors 去处理只接近他们的数据， 这样可以降低带宽。 每个executors 的 core 执行它自己的数据

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530105957-each-executor%E2%80%99s-core-gets-a-partition-of-data-to-work-on.png)

```py
# In Python
log_df = spark.read.text("path_to_large_text_file").repartition(8)
print(log_df.rdd.getNumPartitions())
```

