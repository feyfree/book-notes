# Strutured Streaming 

讨论Spark 的流式计算API

## Evolution of the Apache Spark Stream Processing Engine

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220612163108-traditional-record-at-a-time-processing-model.png)

The processing pipeline is composed of a directed graph of nodes, as shown in Figure 8-1; each node continuously receives one record at a time, processes it, and then forwards the generated record(s) to the next node in the graph. This processing model can achieve very low latencies—that is, an input record can be processed by the pipeline and the resulting output can be generated within milliseconds. However, this model is not very efficient at recovering from node failures and straggler nodes (i.e., nodes that are slower than others); it can either recover from a failure very fast with a lot of extra failover resources, or use minimal extra resources but recover slowly

传统的 record-at-time 容错机制不够,  recover 缓慢

### The Advent of Micro-Batch Stream Processing

Spark Streaming (DStream)

It introduced the idea of micro-batch stream processing, where the streaming computation is modeled as a continuous series of small, map/reduce-style batch processing jobs (hence, “micro-batches”) on small chunks of the stream data

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220612163506-micro-batch-processing-model-in-structured-streaming.png)

As shown here, Spark Streaming divides the data from the input stream into, say, 1- second micro-batches. Each batch is processed in the Spark cluster in a distributed manner with small deterministic tasks that generate the output in micro-batches. Breaking down the streaming computation into these small tasks gives us two advan‐ tages over the traditional, continuous-operator model:

* Spark’s agile task scheduling can very quickly and efficiently recover from failures and straggler executors by rescheduling one or more copies of the tasks on any of the other executors. 

​		Spark 的敏捷任务调度可以通过重新调度任何其他执行器上的一个或多个任务副本, 来非常快速有效地从故障和落后的执行器中恢复

* The deterministic nature of the tasks ensures that the output data is the same no matter how many times the task is reexecuted. This crucial characteristic enables Spark Streaming to provide end-to-end exactly-once processing guarantees, that is, the generated output results will be such that every input record was processed exactly once.

  任务的确定性确保无论任务被重新执行多少次，输出数据都是相同的。 这一关键特性使 Spark Streaming 能够提供端到端的Exactly-once 处理保证，即生成的输出结果将使得每个输入记录都被处理一次

这种高效的容错确实是以延迟为代价的——微批处理模型无法实现毫秒级的延迟； 它通常会达到几秒的延迟（在某些情况下低至半秒）。 但是，我们观察到，对于绝大多数流处理用例，微批处理的好处处理超过了这种延迟(可以比喻为second-scale latency)的缺点。 这是因为大多数流式传输管道至少具有以下特征之一:

1. pipeline 不需要低于几秒的延迟。 例如，当流输出仅由每小时作业读取时，生成具有亚秒级延迟的输出是没有用的
2. pipeline 的其他部分存在较大的延迟。 例如，如果将传感器写入 Apache Kafka（用于摄取数据流的系统）进行批处理以实现更高的吞吐量，那么下游处理系统中的任何优化都无法使端到端延迟低于批处理延迟.

Furthermore, the DStream API was built upon Spark’s batch RDD API. Therefore, DStreams had the same functional semantics and fault-tolerance model as RDDs. Spark Streaming thus proved that it is possible for a single, unified processing engine to provide consistent APIs and semantics for batch, interactive, and streaming workloads. This fundamental paradigm shift in stream processing propelled Spark Streaming to become one of the most widely used open source stream processing engines.

### Lessons Learned from Spark Streaming (DStreams)

除了这些优势, DStream API 还有一些缺点. 下面列举的是一些确定的需要改善的地方

1. Lack of a single API for batch and stream processing

尽管DStreams 和 RDDs 具有一致的API (相同的操作, 相同的语义), 开发者还是需要重写他们的代码去使用不同的类, 去将batch jobs 转化成为 streaming jobs

1. Lack of separation between logical and physical plans

Spark Streaming 依据开发者指明的操作顺序执行 DStream 操作. 由于开发人员有效地指定了确切的物理计划，因此没有自动优化的余地，开发人员必须手动优化他们的代码以获得最佳性能

1. Lack of native support for event-time windows

DStream 仅根据 Spark Streaming 接收每条记录的时间（称为处理时间）来定义窗口操作。 但是，许多用例需要根据生成记录的时间（称为事件时间）而不是接收或处理记录的时间来计算窗口聚合。 由于缺乏对事件时间窗口的原生支持，开发人员很难使用 Spark Streaming 构建此类管道。

### The Philosophy of Structured Streaming

一个理念, 写streaming pipeline 和 写batch pipeline 一样简单. 理念如下

1. A single, unified programming model and interface for batch and stream processing
2. A broader definition of stream processing

## The Programming Model of Structured Streaming

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220613094809-the-structured-streaming-programming-model-data-stream-as-an-unbounded-table.png)

Every new record received in the data stream is like a new row being appended to the unbounded input table. Structured Streaming will not actually retain all the input, but the output produced by Structured Streaming until time T will be equivalent to having all of the input until T in a static, bounded table and running a batch job on the table.

结构化流式处理实际上不会保留所有输入，但结构化流式处理在时间 T 之前产生的输出将等价于将所有输入直到 T 保存在一个静态的有界表中并在该表上运行批处理作业.

As shown in Figure 8-4, the developer then defines a query on this conceptual input table, as if it were a static table, to compute the result table that will be written to an output sink. Structured Streaming will automatically convert this batch-like query to a streaming execution plan. This is called incrementalization: Structured Streaming figures out what state needs to be maintained to update the result each time a record arrives. Finally, developers specify triggering policies to control when to update the results. Each time a trigger fires, Structured Streaming checks for new data (i.e., a new row in the input table) and incrementally updates the result

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220613095630-the-structured-streaming-processing-model.png)

Structured Streaming provides three output modes:

1. Append Mode
2. Update Mode
3. Complete Mode

Unless complete mode is specified, the result table will not be fully materialized by Structured Streaming. Just enough information (known as “state”) will be maintained to ensure that the changes in the result table can be computed and the updates can be output.

## The Fundamentals of a Structured Streaming Query

In this section, we are going to cover some high-level concepts that you’ll need to understand to develop Structured Streaming queries. We will first walk through the key steps to define and start a streaming query, then we will discuss how to monitor the active query and manage its life cycle

### Five Steps to Define a Streaming Query

1. Define input sources

2. Transform data

   1. Stateless transformations : 不依赖previous rows 比如 select() filter() map() 等等
   2. Stateful transformations: 以来previous rows 比如 count()

3. Define output sink and output mode

   * Output writing details (where and how to write the output)
   * Processing details (how to process data and how to recover from failures)

4. Step 4: Specify processing details

   **Triggering Details** 

   1. Default (When the trigger is not explicitly specified, then by default, the streaming query executes data in micro-batches where the next micro-batch is trig‐ gered as soon as the previous micro-batch has completed)
   2. Processing time with trigger interval (You can explicitly specify the ProcessingTime trigger with an interval, and the query will trigger micro-batches at that fixed interval)
   3. Once (In this mode, the streaming query will execute exactly one micro-batch—it processes all the new data available in a single batch and then stops itself. This is useful when you want to control the triggering and processing from an external scheduler that will restart the query using any custom schedule (e.g., to control cost by only executing a query once per day)
   4. Continuous (This is an experimental mode (as of Spark 3.0) where the streaming query will process data continuously instead of in micro-batches. While only a small subset of DataFrame operations allow this mode to be used, it can pro‐ vide much lower latency (as low as milliseconds) than the micro-batch trig‐ ger modes. Refer to the latest Structured Streaming Programming Guide for the most up-to-date information)

   **Checkpoint location**

   This is a directory in any HDFS-compatible filesystem where a streaming query saves its progress information—that is, what data has been successfully pro‐ cessed. Upon failure, this metadata is used to restart the failed query exactly where it left off. Therefore, setting this option is necessary for failure recovery with exactly-once guarantees

5. Step 5: Start the query

### Under the Hood of an Active Streaming Query

Once the query starts, the following sequence of steps transpires in the engine, as depicted in Figure 8-5. The DataFrame operations are converted into a logical plan, which is an abstract representation of the computation that Spark SQL uses to plan a query:

1. Spark SQL analyzes and optimizes this logical plan to ensure that it can be exe‐ cuted incrementally and efficiently on streaming data
2. Spark SQL starts a background thread that continuously executes the following loop (**not for continuous mode**)
   1. Based on the configured trigger interval, the thread checks the streaming sources for the availability of new data
   2. If available, the new data is executed by running a micro-batch. From the optimized logical plan, an optimized Spark execution plan is generated that reads the new data from the source, incrementally computes the updated result, and writes the output to the sink according to the configured output mode
   3. For every micro-batch, the exact range of data processed (e.g., the set of files or the range of Apache Kafka offsets) and any associated state are saved in the configured checkpoint location so that the query can deterministically reproc‐ ess the exact range if needed
3. This loop continues until the query is terminated, which can occur for one of the following reasons:
   1. A failure has occurred in the query (either a processing error or a failure in the cluster).
   2. The query is explicitly stopped using streamingQuery.stop().
   3. If the trigger is set to Once, then the query will stop on its own after executing a single micro-batch containing all the available data

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220613102125-incremental-execution-of-streaming-queries.png)

### Recovering from Failures with Exactly-Once Guarantees

To restart a terminated query in a completely new process, you have to create a new SparkSession, redefine all the DataFrames, and start the streaming query on the final result using the same checkpoint location as the one used when the query was started the first time. For our word count example, you can simply reexecute the entire code snippet shown earlier, from the definition of spark in the first line to the final start() in the last line.

The checkpoint location must be the same across restarts because this directory con‐ tains the unique identity of a streaming query and determines the life cycle of the query. If the checkpoint directory is deleted or the same query is started with a differ‐ ent checkpoint directory, it is like starting a new query from scratch. Specifically, checkpoints have record-level information (e.g., Apache Kafka offsets) to track the data range the last incomplete micro-batch was processing. The restarted query will use this information to start processing records precisely after the last successfully completed micro-batch. If the previous query had planned a micro-batch but had ter‐ minated before completion, then the restarted query will reprocess the same range of data before processing new data. Coupled with Spark’s deterministic task execution, the regenerated output will be the same as it was expected to be before the restart.

Structured Streaming can ensure end-to-end exactly-once guarantees (that is, the out‐ put is as if each input record was processed exactly once) when the following conditions have been satisfied:

**Replayable streaming sources**

The data range of the last incomplete micro-batch can be reread from the source

**Deterministic computations**

All data transformations deterministically produce the same result when given the same input data.

**Idempotent streaming sink**

The sink can identify reexecuted micro-batches and ignore duplicate writes that may be caused by restarts.

**DataFrame transformations**

**Source and sink options**

**Processing details**

### Monitoring an Active Query

An important part of running a streaming pipeline in production is tracking its health. Structured Streaming provides several ways to track the status and processing metrics of an active query.

**Querying current status using StreamingQuery**

**Get current metrics using StreamingQuery**

**Get current status using StreamingQuery.status().**

**Publishing metrics using Dropwizard Metrics**

**Publishing metrics using custom StreamingQueryListeners**

1. Define your custom listener

```scala
// In Scala
import org.apache.spark.sql.streaming._
val myListener = new StreamingQueryListener() {
 override def onQueryStarted(event: QueryStartedEvent): Unit = {
 println("Query started: " + event.id)
 }
 override def onQueryTerminated(event: QueryTerminatedEvent): Unit = {
 println("Query terminated: " + event.id)
 }
 override def onQueryProgress(event: QueryProgressEvent): Unit = {
 println("Query made progress: " + event.progress)
 }
}
```

2. Add your listener to the SparkSession before starting the query:

```scala
// In Scala
spark.streams.addListener(myListener)
```

## Streaming Data Sources and Sinks

Now that we have covered the basic steps you need to express an end-to-end Struc‐ tured Streaming query, let’s examine how to use the built-in streaming data sources and sinks. As a reminder, you can create DataFrames from streaming sources using SparkSession.readStream() and write the output from a result DataFrame using DataFrame.writeStream(). In each case, you can specify the source type using the method format(). We will see a few concrete examples later.

### Files

**Reading from files**

```scala
// In Scala
import org.apache.spark.sql.types._
val inputDirectoryOfJsonFiles = ...
val fileSchema = new StructType()
 .add("key", IntegerType)
 .add("value", IntegerType)
val inputDF = spark.readStream
 .format("json")
 .schema(fileSchema)
 .load(inputDirectoryOfJsonFiles)
```

Tips:

*  All the files must be of the same format and are expected to have the same schema. For example, if the format is "json", all the files must be in the JSON format with one JSON record per line. The schema of each JSON record must match the one specified with readStream(). Violation of these assumptions can lead to incorrect parsing (e.g., unexpected null values) or query failures. 
*  Each file must appear in the directory listing atomically—that is, the whole file must be available at once for reading, and once it is available, the file cannot be updated or modified. This is because Structured Streaming will process the file when the engine finds it (using directory listing) and internally mark it as pro‐ cessed. Any changes to that file will not be processed. 
*  When there are multiple new files to process but it can only pick some of them in the next micro-batch (e.g., because of rate limits), it will select the files with the earliest timestamps. Within the micro-batch, however, there is no predefined order of reading of the selected files; all of them will be read in parallel.

**Writing to files**

```scala
// In Scala
val outputDir = ...
val checkpointDir = ...
val resultDF = ...
val streamingQuery = resultDF
 .writeStream
 .format("parquet")
 .option("path", outputDir)
 .option("checkpointLocation", checkpointDir)
 .start()
```

Tips:

* Structured Streaming achieves end-to-end exactly-once guarantees when writing to files by maintaining a log of the data files that have been written to the direc‐ tory. This log is maintained in the subdirectory ***_spark_metadata***. Any Spark query on the directory (not its subdirectories) will automatically use the log to read the correct set of data files so that the exactly-once guarantee is maintained (i.e., no duplicate data or partial files are read). Note that other processing engines may not be aware of this log and hence may not provide the same guarantee. 
*  If you change the schema of the result DataFrame between restarts, then the out‐ put directory will have data in multiple schemas. These schemas have to be rec‐ onciled when querying the directory

### Apache Kafka

**Read**

```scala
// In Scala
val inputDF = spark
 .readStream
 .format("kafka")
 .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
 .option("subscribe", "events")
 .load()
```

**Write**

```scala
// In Scala
val counts = ... // DataFrame[word: string, count: long]
val streamingQuery = counts
 .selectExpr(
 "cast(word as string) as key",
 "cast(count as string) as value")
 .writeStream
 .format("kafka")
 .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
 .option("topic", "wordCounts")
 .outputMode("update")
 .option("checkpointLocation", checkpointDir)
 .start()
```

### Custom Streaming Sources and Sinks

**Writing to any storage system**

1. **Using foreachBatch().** 
   1. Reuse existing batch data sources
   2. Write to multiple locations
   3. Apply additional DataFrame operations
2. **Using foreach()**

**Reading from any storage system**

略

## Data Transformations

These operations are broadly classified into stateless and stateful operations. We will define each type of operation and explain how to identify which operations are stateful;

### Incremental Execution and Streaming State

之前讲过,   Spark SQL 的 Catalyst optimizer 会将所有的DataFrame 的操作 转化成为优化过的 logical plan.

The Spark SQL planner, which decides how to execute a logical plan, recognizes that this is a streaming logical plan that needs to operate on continuous data streams. Accordingly, instead of converting the logical plan to a one-time physical execution plan, the planner generates a continuous sequence of execution plans. Each execution plan updates the final result DataFrame incrementally—that is, the plan processes only a chunk of new data from the input streams and possibly some intermediate, partial result computed by the previous execution plan



Each execution is considered as a micro-batch, and the partial intermediate result that is communicated between the executions is called the streaming “state.” Data‐ Frame operations can be broadly classified into stateless and stateful operations based on whether executing the operation incrementally requires maintaining a state

### Stateless Transformations

All projection operations (e.g., select(), explode(), map(), flatMap()) and selec‐ tion operations (e.g., filter(), where()) process each input record individually without needing any information from previous rows. This lack of dependence on prior input data makes them stateless operations.

只有无状态操作的流式查询支持Append和Update输出模式，但不支持Complete模式。 这是有道理的：由于此类查询的任何已处理输出行都不能被任何未来数据修改，因此可以将其写入所有附加模式下的流式接收器（包括仅附加的接收器，如任何格式的文件）。 另一方面，此类查询自然不会跨输入记录组合信息，因此可能不会减少结果中的数据量。 不支持Complete Mode，因为存储不断增长的结果数据通常成本很高。 这与有状态转换形成鲜明对比，我们将讨论
下一个

### Stateful Transformations

The simplest example of a stateful transformation is DataFrame.groupBy().count(), which generates a running count of the number of records received since the beginning of the query.

**Distributed and fault-tolerant state management**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220614134443-distributed-state-management-in-structured-streaming.png)

To summarize, for all stateful operations, Structured Streaming ensures the correctness of the operation by automatically saving and restoring the state in a distributed manner. Depending on the stateful operation, all you may have to do is tune the state cleanup policy such that old keys and values can be automatically dropped from the cached state

**Types of stateful operations**

The essence of streaming state is to retain summaries of past data. Sometimes old summaries need to be cleaned up from the state to make room for new summaries. Based on how this is done, we can distinguish two types of stateful operations:

*Managed stateful operations*

* Streaming aggregations
* Stream-stream joins
* Streaming deduplication

*Unmanaged stateful operations*

These operations let you define your own custom state cleanup logic. The operations in this category are:

* MapGroupsWithState
* FlatMapGroupsWithState

## Stateful Streaming Aggregations

Structured Streaming can incrementally execute most DataFrame aggregation opera‐ tions. You can aggregate data by keys (e.g., streaming word count) and/or by time (e.g., count records received every hour).

### Aggregations Not Based on Time

Aggregations not involving time can be broadly classified into two categories:

***Global aggregations***

Aggregations across all the data in the stream. For example, say you have a stream of sensor readings as a streaming DataFrame named sensorReadings. You can calculate the running count of the total number of readings received with the following query:

```scala
// In Scala
val runningCount = sensorReadings.groupBy().count()
```

Tips:

**You cannot use direct aggregation operations like DataFrame.count() and Dataset.reduce() on streaming DataFrames**. This is because, for static DataFrames, these operations immediately return the final computed aggregates, whereas for streaming DataFrames the aggregates have to be continuously updated. Therefore, you have to always use **DataFrame.groupBy() or Dataset.groupByKey() for aggregations on streaming DataFrames**.

***Grouped aggregations***

Grouped aggregations

Aggregations within each group or key present in the data stream. For example, if sensorReadings contains data from multiple sensors, you can calculate the running average reading of each sensor (say, for setting up a baseline value for each sensor) with the following:

```scala
// In Scala
val baselineValues = sensorReadings.groupBy("sensorId").mean("value")
```

***All built-in aggregation functions***

sum(), mean(), stddev(), countDistinct(), collect_set(), approx_count_dis tinct(), etc. Refer to the API documentation (Python and Scala) for more details.

***Multiple aggregations computed together***

You can apply multiple aggregation functions to be computed together in the fol‐ lowing manner:

```scala
// In Scala
import org.apache.spark.sql.functions.*
val multipleAggs = sensorReadings
 .groupBy("sensorId")
 .agg(count("*"), mean("value").alias("baselineValue"),
 collect_set("errorCode").alias("allErrorCodes"))
```

***User-defined aggregation functions***

All user-defined aggregation functions are supported. See the Spark SQL pro‐ gramming guide for more details on untyped and typed user-defined aggregation functions

### Aggregations with Event-Time Windows

```scala
// In Scala
import org.apache.spark.sql.functions.*
sensorReadings
 .groupBy("sensorId", window("eventTime", "5 minute"))
 .count()
```

The key thing to note here is the window() function, which allows us to express the five-minute windows as a dynamically computed grouping column. When started, this query will effectively do the following for each sensor reading: 

* Use the eventTime value to compute the five-minute time window the sensor reading falls into. 
* Group the reading based on the composite group (\<computerd window>, SensorId). 
* Update the count of the composite group.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220614190402-mapping-of-event-time-to-tumbling-windows.png)

```scala
// In Scala
sensorReadings
 .groupBy("sensorId", window("eventTime", "10 minute", "5 minute"))
 .count()
```

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220614190629-mapping-of-event-time-to-multiple-overlapping-windows.png)

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220614191015-updated-counts-in-the-result-table-after-each-five-minute-trigger.png)

一个新的windows 创建, 但是老的windows 仍然占用内存, 等待一些滞后的数据更新他们. 实际上, Query 很难知道滞后的数据到底要滞后多久. 

所以 Too old to receive updates 问题出现了, 如何将 too old 的数据 丢弃掉. 

watermark 出现了

#### Handling late data with watermarks

watermark 被定义为 是在event time 中移动的threshold,  表明的是在处理数据时候能查询到的最大的 event time

The trailing gap, known as the **watermark delay**, defines how long the engine will wait for late data to arrive

```scala
// In Scala
sensorReadings
 .withWatermark("eventTime", "10 minutes")
 .groupBy("sensorId", window("eventTime", "10 minutes", "5 minute"))
 .mean("value")
```

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220614191938-spark-late-data-watermark-illustration.png)

x 轴是处理时间

y 轴是event time

理解上述图的含义

1. 相当于是阶梯 (watermark)上面的数据都能被处理
2. 阶梯下面的数据 实际上too late 的数据,会被丢弃

#### Semantic guarantees with watermarks.

在结束关于watermark的这一部分之前，让我们考虑一下watermark提供的精确语义保证。 10 分钟的watermark可确保引擎永远不会丢弃与输入数据中看到的最新事件时间相比延迟少于 10 分钟的任何数据。 但是，保证只在一个方向上是严格的。 延迟超过 10 分钟的数据不能保证被丢弃——也就是说，它可能会被聚合。延迟超过 10 分钟的输入记录是否会被实际聚合取决于接收记录的确切时间以及何时它被触发的微批处理

#### Supported output modes

Update mode

在这种模式下，每个mirco-batch将只输出聚合更新的行。 此模式可用于所有类型的聚合。 特别是对于时间窗口聚合，watermark将确保定期清理状态。 这是使用流式聚合运行查询的最有用和最有效的模式。 但是，您不能使用此模式将聚合写入append-only sinks，例如任何基于文件的格式，如 Parquet 和 ORC（除非您使用 Delta Lake，我们将在下一章讨论）

Complete mode

在这种模式下，每个mirco-batch都将输出所有更新的聚合，无论它们的年龄或是否包含更改。 虽然此模式可用于所有类型的聚合，但对于时间窗口聚合，使用完整模式意味着即使指定了watermark也不会清除状态。 输出所有聚合需要所有过去的state，因此必须保留聚合数据。即使已定义watermark。 谨慎地在时间窗口聚合上使用此模式，因为这可能导致state大小和内存使用量的无限增加

Append mode

*This mode can be used only with aggregations on event-time windows and with watermarking enabled*

回想一下append mode不允许以前的输出结果改变。 对于任何没有watermark的聚合，每个聚合都可以用任何未来的数据进行更新，因此这些不能以append mode输出。 只有在事件时间窗口上的聚合上启用watermark时，查询才知道聚合何时不会进一步更新。因此，append mode仅在水印时才输出每个键及其最终聚合值，而不是输出更新的行 确保聚合不会再次更新。 此模式的优点是它允许您将聚合写入append-only的sinks（例如，文件）。 缺点是输出会被watermark持续时间延迟——查询必须等待尾随watermark超过一个键的时间窗口，然后才能完成它的聚合。

## Streaming Joins

Structured Streaming supports joining a streaming Dataset with another static or streaming Dataset.

### Stream–Static Joins

```scala
// In Scala
// Static DataFrame [adId: String, impressionTime: Timestamp, ...]
// reading from your static data source
val impressionsStatic = spark.read. ...
// Streaming DataFrame [adId: String, clickTime: Timestamp, ...]
// reading from your streaming source
val clicksStream = spark.readStream. ...
// In Scala inner join
val matched = clicksStream.join(impressionsStatic, "adId")

// In Scala left outer join
val matched = clicksStream.join(impressionsStatic, Seq("adId"), "leftOuter")
```

注意点

1. static join is stateless operation
2. static DataFrame 如果被 重复读取的话, 考虑使用cache, 用来加速
3. 如果定义了静态 DataFrame 的数据源中的底层数据发生了变化，流式查询是否能看到这些变化取决于datasource的具体行为。 例如，如果静态 DataFrame 是在文件上定义的，那么在重新启动流式查询之前，不会拾取对这些文件的更改（例如，追加）

### Stream-Stream Joins

```scala
// In Scala
// Streaming DataFrame [adId: String, impressionTime: Timestamp, ...]
val impressions = spark.readStream. ...
// Streaming DataFrame[adId: String, clickTime: Timestamp, ...]
val clicks = spark.readStream. ...
val matched = impressions.join(clicks, "adId")
```

The engine will buffer all clicks and impressions as state, and will generate a matching impression-and-click as soon as a received click matches a buffered impression (or vice versa, depending on which was received first).

注意两点 (to limit the streaming state maintained by stream-stream joins)

1. What is the maximum time range between the generation of the two events at their respective sources?
2. What is the maximum duration an event can be delayed in transit between the source and the processing engine?

两个event 从各自的datasource 来的最大时间间隔

process 等待 event 从datasource 来的最大时间间隔

These delay limits and event-time constraints can be encoded in the DataFrame operations using watermarks and time range conditions. In other words, you will have to do the following additional steps in the join to ensure state cleanup:

1. Define watermark delays on both inputs, such that the engine knows how delayed the input can be (similar to with streaming aggregations).
2. Define a constraint on event time across the two inputs, such that the engine can figure out when old rows of one input are not going to be required (i.e., will not satisfy the time constraint) for matches with the other input. This constraint can be defined in one of the following ways:
   1. Time range join conditions (e.g., join condition = "leftTime BETWEEN rightTime AND rightTime + INTERVAL 1 HOUR")
   2. Join on event-time windows (e.g., join condition = "leftTimeWindow = rightTimeWindow")

#### Inner joins with optional watermarking

```scala
// In Scala
// Define watermarks
val impressionsWithWatermark = impressions
 .selectExpr("adId AS impressionAdId", "impressionTime")
 .withWatermark("impressionTime", "2 hours ")

val clicksWithWatermark = clicks
 .selectExpr("adId AS clickAdId", "clickTime")
 .withWatermark("clickTime", "3 hours")

// Inner join with time range conditions
impressionsWithWatermark.join(clicksWithWatermark,
 expr("""
 clickAdId = impressionAdId AND
 clickTime BETWEEN impressionTime AND impressionTime + interval 1 hour"""))
```

1. impressions 会被buffered 最多 4 个小时  (3 + 1)
2. clicks 最多会被buffered 2个小时, Conversely, clicks need to be buffered for at most two hours (in event time), as a two-hour-late impression may match with a click received two hours ago.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220616185907-structured-streaming-automatically-calculates-thresholds-for-state-cleanup-using-watermark-delays-and-time-range-conditions.png)

inner joins 需要注意的地方

1. 对于inner joins, 指明 watermarking 还有 event-time 的约束都不是必须项目 (optional).  In other words, at the risk of potentially unbounded state, you may choose not to specify them. Only when both are specified will you get state cleanup.
2. Similar to the guarantees provided by watermarking on aggregations, a watermark delay of two hours guarantees that the engine will never drop or not match any data that is less than two hours delayed, but data delayed by more than two hours may or may not get processed. (watermark 延迟之内的数据不会丢弃, 超过watermark 的延迟的可能不会被执行)

#### Outer joins with watermarking

outer join 像是去并集, 比如是left outer, 是左边匹配右边, 如果右边匹配不上, 默认为NULL

```scala
// In Scala
// Left outer join with time range conditions
impressionsWithWatermark.join(clicksWithWatermark,
 expr("""
 clickAdId = impressionAdId AND
 clickTime BETWEEN impressionTime AND impressionTime + interval 1 hour"""),
 "leftOuter") // Only change: set the outer join type
```

注意点

1. Unlike with inner joins, the watermark delay and event-time constraints are not optional for outer joins. This is because for generating the NULL results, the engine must know when an event is not going to match with anything else in the future. For correct outer join results and state cleanup, the watermarking and event-time constraints must be specified.(意思是 outer join 必须要指明watermark 还有 event time 限制)
2. Consequently, the outer NULL results will be generated with a delay as the engine has to wait for a while to ensure that there neither were nor would be any matches. This delay is the maximum buffering time (with respect to event time) calculated by the engine for each event as discussed in the previous section (i.e., four hours for impressions and two hours for clicks).

## Arbitrary Stateful Computations

The operation mapGroupsWithState() and its more flexible counterpart flatMapGroupsWithState() are designed for such complex analytical use cases 

Spark 3.0 中, 这两个api 只在 Java 和 Scala 里面有

用法略

## Performance Tuning

1. Cluster resource provisioning.

   allocation should be done based on the nature of the streaming queries: stateless queries usually need more cores, and stateful queries usually need more memory

2. Number of partitions for shuffles

对于structured stream查询，shuffle 分区的数量通常需要设置得比大多数批处理查询低得多——过多地划分计算会增加开销并降低脱兔量。 此外，由于有状态操作的shuffle 由于 checkpointing 而具有显着更高的任务开销。 因此，对于stateful和触发间隔为几秒到几分钟的流式查询，建议将 shuffle 分区的数量从默认值 200 调整到最多分配核心数的两到三倍。

3. Setting source rate limits for stability

   1. Setting the limit too low can cause the query to underutilize allocated resour‐ ces and fall behind the input rate.
   2. Limits do not effectively guard against sustained increases in input rate. While stability is maintained, the volume of buffered, unprocessed data will grow indefinitely at the source and so will the end-to-end latencies

4. Multiple streaming queries in the same Spark application

   1. Executing each query continuously uses resources in the Spark driver (i.e., the JVM where it is running). This limits the number of queries that the driver can execute simultaneously. Hitting those limits can either bottleneck the task scheduling (i.e., underutilizing the executors) or exceed memory limits
   2. You can ensure fairer resource allocation between queries in the same context by setting them to run in separate scheduler pools. Set the SparkContext’s thread-local property spark.scheduler.pool to a different string value for each stream:

   ```scala
   // In Scala
   // Run streaming query1 in scheduler pool1
   spark.sparkContext.setLocalProperty("spark.scheduler.pool", "pool1")
   df.writeStream.queryName("query1").format("parquet").start(path1)
   // Run streaming query2 in scheduler pool2
   spark.sparkContext.setLocalProperty("spark.scheduler.pool", "pool2")
   df.writeStream.queryName("query2").format("parquet").start(path2)
   ```

   