# Optimizing and Tuning Spark Applications

Spark 系统调优， 观察 join 策略， Spark UI 发现优化的线索

## Optimizing and Tuning Spark for Efficiency

[Spark Configuration](https://spark.apache.org/docs/latest/configuration.html)

通过参数调优

### Viewing and Setting Apache Spark Configurations

三种方法， 传入或者设置参数

1. 通过配置文件

2. 通过spark-submit 参数提交这种形式， 或者是通过 scala 或者 python 构造参数引入

   ```shell
   spark-submit --conf spark.sql.shuffle.partitions=5 --conf
   "spark.executor.memory=2g" --class main.scala.chapter7.SparkConfig_7_1 jars/mainscala-chapter7_2.12-1.0.jar
   ```

   ```scala
   // In Scala
   import org.apache.spark.sql.SparkSession
   def printConfigs(session: SparkSession) = {
    // Get conf
    val mconf = session.conf.getAll
    // Print them
    for (k <- mconf.keySet) { println(s"${k} -> ${mconf(k)}\n") }
   }
   def main(args: Array[String]) {
   // Create a session
   val spark = SparkSession.builder
    .config("spark.sql.shuffle.partitions", 5)
    .config("spark.executor.memory", "2g")
    .master("local[*]")
    .appName("SparkConfig")
    .getOrCreate()
   printConfigs(spark)
   spark.conf.set("spark.sql.shuffle.partitions",
    spark.sparkContext.defaultParallelism)
   println(" ****** Setting Shuffle Partitions to Default Parallelism")
   printConfigs(spark)
   }
   ```

3. 通过spark-shell

   ```shell
   // In Scala
   // mconf is a Map[String, String]
   scala> val mconf = spark.conf.getAll
   ...
   scala> for (k <- mconf.keySet) { println(s"${k} -> ${mconf(k)}\n") }
   spark.driver.host -> 10.13.200.101
   spark.driver.port -> 65204
   spark.repl.class.uri -> spark://10.13.200.101:65204/classes
   spark.jars ->
   spark.repl.class.outputDir -> /private/var/folders/jz/qg062ynx5v39wwmfxmph5nn...
   spark.app.name -> Spark shell
   spark.submit.pyFiles ->
   spark.ui.showConsoleProgress -> true
   spark.executor.id -> driver
   spark.submit.deployMode -> client
   spark.master -> local[*]
   spark.home -> /Users/julesdamji/spark/spark-3.0.0-preview2-bin-hadoop2.7
   spark.sql.catalogImplementation -> hive
   spark.app.id -> local-1580144503745
   
   // In Scala
   spark.sql("SET -v").select("key", "value").show(5, false)
   # In Python
   spark.sql("SET -v").select("key", "value").show(n=5, truncate=False)
   ```

检测参数是否可修改

`spark.conf.isModifiable("")` will return true or false

如果可修改

```shell
// In Scala
scala> spark.conf.get("spark.sql.shuffle.partitions")
res26: String = 200
scala> spark.conf.set("spark.sql.shuffle.partitions", 5)
scala> spark.conf.get("spark.sql.shuffle.partitions")
res28: String = 5
# In Python
>>> spark.conf.get("spark.sql.shuffle.partitions")
'200'
>>> spark.conf.set("spark.sql.shuffle.partitions", 5)
>>> spark.conf.get("spark.sql.shuffle.partitions")
'5'
```

读取的步骤

1. 配置文件
2. spark-submit 参数
3. 应用配置

通过SparkConf 对象配置的属性优先级最高；其次是对spark-submit 或 spark-shell通过flags配置；最后是spark-defaults.conf文件中的配置

### Scaling Spark for Large Workloads

通过影响三个组件

1. spark driver
2. spark executor
3. shuffle service running on the executor

**Static versus dynamic resource allocation**

动态分配， Spark 会根据当前的使用情况， 对资源的分配进行调整， 提高整体的利用率

默认的话 `spark.dynamicAllocation.enabled` 是 false

可以通过编码配置

```scala
spark.dynamicAllocation.enabled true
spark.dynamicAllocation.minExecutors 2
spark.dynamicAllocation.schedulerBacklogTimeout 1m
spark.dynamicAllocation.maxExecutors 20
spark.dynamicAllocation.executorIdleTimeout 2min
```

**Configuring Spark executors’ memory and the shuffle service**

开启动态分配还不够， 还需要了解executor 的内存分布， 以及当前spark 的使用情况， 这样可以避免executor 面临内存资源不够， 或者是被JVM GC 影响

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220607164936-spark-executor-memory-layout.png)

The Spark documentation advises that this will work for most cases, but you can adjust what fraction of spark.executor.memory you want either section to use as a baseline. When storage memory is not being used, Spark can acquire it for use in execution memory for execution purposes, and vice versa.

1. Execution memory is used for Spark shuffles, joins, sorts, and aggregations
2. storage memory is primarily used for caching user data structures and partitions derived from DataFrames

During **map and shuffle** operations, Spark writes to and reads from the local disk’s shuffle files, so there is heavy I/O activity. This can result in a bottleneck, because the default configurations are suboptimal for large-scale Spark jobs. Knowing what configurations to tweak can mitigate this risk during this phase of a Spark job.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220607165417-spark-configurations-to-tweak-for-i/o-during-map-and-shuffle-operations.png)

**Maximizing Spark parallelism**

Spark is embarrassingly efficient at processing its tasks in parallel。

大规模的负载下，一个Spark Job 可能有很多Stages， 每个Stage 下面可能有好多task。Spark will at best schedule a thread per task per core, and each task will process a distinct partition

**How partitions are created**

The size of a partition in Spark is dictated by spark.sql.files.maxPartitionBytes. The default is 128 MB.

你可以降低这个数值， 但是可能会带来 "小文件问题"， 反而会增加IO， 降低性能

调用DF API的时候， 比如创建一个大的DF 或者 从磁盘读大文件的时候， 也会带来partitions, 你也可以指定数量

```scala
// In Scala
val ds = spark.read.textFile("../README.md").repartition(16)
ds: org.apache.spark.sql.Dataset[String] = [value: string]

ds.rdd.getNumPartitions
res5: Int = 16

val numDF = spark.range(1000L * 1000 * 1000).repartition(16)
numDF.rdd.getNumPartitions

numDF: org.apache.spark.sql.Dataset[Long] = [id: bigint]
res12: Int = 16
```

Finally, shuffle partitions are created during the shuffle stage. By default, the number of shuffle partitions is set to 200 in **spark.sql.shuffle.partitions**. You can adjust this number depending on the size of the data set you have, to reduce the amount of small partitions being sent across the network to executors’ tasks.

负载比较小的话， 可以调小一下 spark.sql.shuffle.partitions （比如可以调成和 executor  cores 数量一样甚至更小）

## Caching and Persistence of Data

Two API calls, cache() and persist(), offer these capabilities

### DataFrame.cache()

DataFrame 可以fractionally cached , partition 不能， 一个partition 必须完整的被cache 或者 不被cache， 不存在部分cache

```scala
// In Scala
// Create a DataFrame with 10M records
val df = spark.range(1 * 10000000).toDF("id").withColumn("square", $"id" * $"id")
df.cache() // Cache the data
df.count() // Materialize the cache
res3: Long = 10000000
Command took 5.11 seconds
df.count() // Now get it from the cache
res4: Long = 10000000
Command took 0.44 seconds
```

当显示调用的时候cache() 的时候， spark 不会立刻去cache全部， 除非你遍历了所有的记录， 比如 （count()）, 这些数据会cache， 如果是take(1) 这种动作， 只会立刻cache 一条

### DataFrame.persist()

persist(StorageLevel.LEVEL) is nuanced, providing control over how your data is cached via StorageLevel.  Table 7-2 summarizes the different storage levels. Data on disk is always serialized using either Java or Kryo serialization

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220607180402-spark-storage-levels.png)

```scala
// In Scala
import org.apache.spark.storage.StorageLevel
// Create a DataFrame with 10M records
val df = spark.range(1 * 10000000).toDF("id").withColumn("square", $"id" * $"id")
df.persist(StorageLevel.DISK_ONLY) // Serialize the data and cache it on disk
df.count() // Materialize the cache
res2: Long = 10000000
Command took 2.08 seconds
df.count() // Now get it from the cache
res3: Long = 10000000
Command took 0.38 seconds
```

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220607182932-spark-persist-demo.png)

Finally, not only can you cache DataFrames, but you can also cache the tables or views derived from DataFrames. This gives them more readable names in the Spark UI. For example:

```scala
// In Scala
df.createOrReplaceTempView("dfTable")
spark.sql("CACHE TABLE dfTable")
spark.sql("SELECT count(*) FROM dfTable").show()
+--------+
|count(1)|
+--------+
|10000000|
+--------+
Command took 0.56 seconds
```

### When to Cache and Persist

Common use cases for caching are scenarios where you will want to access a large data set repeatedly for queries or transformations. Some examples include: 

* DataFrames commonly used during iterative machine learning training 

* DataFrames accessed commonly for doing frequent transformations during ETL or building data pipelines

### When Not to Cache and Persist

Not all use cases dictate the need to cache. Some scenarios that may not warrant cach‐ ing your DataFrames include: 

* DataFrames that are too big to fit in memory 

* An inexpensive transformation on a DataFrame not requiring frequent use, regardless of size

As a general rule you should use memory caching judiciously, as it can incur resource costs in serializing and deserializing, depending on the StorageLevel used.

## A Family of Spark Joins

常规的join 类别

1. inner join
2. outer join
3. left join
4. right join

Spark 有五种join 策略

1. the broadcast hash join (BHJ)
2. shuffle hash join (SHJ)
3. shuffle sort merge join (SMJ)
4. broadcast nested loop join (BNLJ)
5. shuffle-and-replicated nested loop join 

主要关注 BHJ 和 SMJ， 这俩是最长用的

