# Apache Spark’s Structured APIs

介绍一些Spark 的高级API， 以及这些API 的设计初衷

## Spark: What’s Underneath an RDD?

The RDD (Resilient Distributed Dataset ) is the most basic abstraction in Spark. There are three vital characteristics associated with an RDD: 

* Dependencies 

* Partitions (with some locality information) 

* Compute function: Partition => Iterator[T]

以上是RDD 的三个特性， 下面分别介绍一下

1. dependencies 表示 Spark 需要哪些输入来构建RDD。需要重建结果的时候， Spark可以通过从这些dependencies中重新创建RDD， 并且复制之前的操作。这样使得RDD更具弹性
2. partitions 使得 Spark 有能力去拆解分布在executors上面的partitions并行化计算任务。In some cases—for example, reading from HDFS—Spark will use locality information to send work to executors close to the data. That way less data is transmitted over the network
3. And finally, an RDD has a compute function that produces an Iterator[T] for the data that will be stored in the RDD.

但是以上的设计会面临一些问题

1. 函数计算对Spark 是不透明的 （Spark 不知道计算函数是什么， 都是当成lambda表达式的）
2. 比如Iterator[T]，Spark 可能仅仅知道在Python里面是个泛型对象
3. 因为Spark不了解特定的类型， Spark 无法进行深入优化， 只能当成一系列的字节对象， 无法进行任何的数据压缩技术

## Structuring Spark

Spark 2.x 版本增加了一些key schemes 来结构化Spark。

其中将数据计算用常用的数据分析的模式去表达， 比如 filtering， selecting, counting, aggregating, averaging and grouping.

通过在 DSL 中使用一组通用运算符进一步缩小了这种特殊性， Spark 支持多语言的这种DSL API， 这种DSL 会告诉Spark 你想对你的数据进行如何计算， 会帮助你构建高效的查询计划。

另外

And the final scheme of order and structure is to allow you to arrange your data in a tabular format, like a SQL table or spreadsheet, with supported structured data types

### Key Merits and Benefits

结构化的好处

1. 可表达性
2. 简洁
3. 组合性
4. 统一性

**expressivity and composability**

使用low-level

```shell
>>> dataRDD = sc.parallelize([("Brooke", 20), ("Denny", 31), ("Jules", 30),
...  ("TD", 35), ("Brooke", 25)])
>>> agesRDD = (dataRDD
...  .map(lambda x: (x[0], (x[1], 1)))
...  .reduceByKey(lambda x, y: (x[0] + y[0], x[1] + y[1]))
...  .map(lambda x: (x[0], x[1][0]/x[1][1])))
```

使用high-level

```shell
>>> from pyspark.sql import SparkSession
>>> from pyspark.sql.functions import avg
>>> spark = (SparkSession
...  .builder
...  .appName("AuthorsAges")
...  .getOrCreate())
>>> data_df = spark.createDataFrame([("Brooke", 20), ("Denny", 31), ("Jules", 30),("TD", 35), ("Brooke", 25)], ["name", "age"])
>>> avg_df = data_df.groupBy("name").agg(avg("age"))
>>> avg_df.show()
+------+--------+
|  name|avg(age)|
+------+--------+
|Brooke|    22.5|
| Denny|    31.0|
| Jules|    30.0|
|    TD|    35.0|
+------+--------+
```

你既可以使用low-level 也可以使用 high-level， 当然大部分情况下使用high-level更简单， 开发更高效， 更易懂

## The DataFrame API

Inspired by pandas DataFrames in structure, format, and a few specific operations, Spark DataFrames are like distributed in-memory tables with named columns and schemas, where each column has a specific data type: integer, string, array, map, real, date, timestamp, etc. To a human’s eye, a Spark DataFrame is like a table

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530165703-the-table-like-format-of-a-dataframe.png)

1. DataFrame 是 immutable的， Spark 会保存transformations 的血统
2. 老版本被保存后， 你可以创建新的DataFrames， 例如增加或者改变column的类型等等，改变名字

### Spark’s Basic Data Types

**scala**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530170234-basic-scala-data-types-in-spark.png)

**python**
![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530170323-basic-python-data-types-in-spark.png)

### Spark’s Structured and Complex Data Types

**scala**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530170508-scala-structured-data-types-in-spark.png)

**python**
![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530170550-python-structured-data-types-in-spark.png)

### Schemas and Creating DataFrames

创建schemas的好处

1. Spark 不需要去推断数据类型
2. Spark 不需要创建额外的任务去推断schema，尤其在特别大的文件中， 这种任务还是很昂贵的
3. schema 可以帮助你排查数据中的错误 （不匹配schema的）

**Two ways to define a schema**

One is to define it programmati‐ cally, and the other is to employ a Data Definition Language (DDL) string, which is much simpler and easier to read.

```shell
// In Scala
import org.apache.spark.sql.types._
val schema = StructType(Array(StructField("author", StringType, false),
 StructField("title", StringType, false),
 StructField("pages", IntegerType, false)))
# In Python
from pyspark.sql.types import *
schema = StructType([StructField("author", StringType(), False),
 StructField("title", StringType(), False),
 StructField("pages", IntegerType(), False)])
```

```scala
// In Scala
val schema = "author STRING, title STRING, pages INT"
# In Python
schema = "author STRING, title STRING, pages INT"
```

一个示例代码

```pyth
from pyspark.sql.types import *
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

# define schema for our data
schema = StructType([
   StructField("Id", IntegerType(), False),
   StructField("First", StringType(), False),
   StructField("Last", StringType(), False),
   StructField("Url", StringType(), False),
   StructField("Published", StringType(), False),
   StructField("Hits", IntegerType(), False),
   StructField("Campaigns", ArrayType(StringType()), False)])

#create our data
data = [[1, "Jules", "Damji", "https://tinyurl.1", "1/4/2016", 4535, ["twitter", "LinkedIn"]],
       [2, "Brooke","Wenig","https://tinyurl.2", "5/5/2018", 8908, ["twitter", "LinkedIn"]],
       [3, "Denny", "Lee", "https://tinyurl.3","6/7/2019",7659, ["web", "twitter", "FB", "LinkedIn"]],
       [4, "Tathagata", "Das","https://tinyurl.4", "5/12/2018", 10568, ["twitter", "FB"]],
       [5, "Matei","Zaharia", "https://tinyurl.5", "5/14/2014", 40578, ["web", "twitter", "FB", "LinkedIn"]],
       [6, "Reynold", "Xin", "https://tinyurl.6", "3/2/2015", 25568, ["twitter", "LinkedIn"]]
      ]
# main program
if __name__ == "__main__":
   #create a SparkSession
   spark = (SparkSession
       .builder
       .appName("Example-3_6")
       .getOrCreate())
   # create a DataFrame using the schema defined above
   blogs_df = spark.createDataFrame(data, schema)
   # show the DataFrame; it should reflect our table above
   blogs_df.show()
   print()
   # print the schema used by Spark to process the DataFrame
   print(blogs_df.printSchema())
   # Show columns and expressions
   blogs_df.select(expr("Hits") * 2).show(2)
   blogs_df.select(col("Hits") * 2).show(2)
   blogs_df.select(expr("Hits * 2")).show(2)
   # show heavy hitters
   blogs_df.withColumn("Big Hitters", (expr("Hits > 10000"))).show()
   print(blogs_df.schema)
```

通过spark-submit 执行， 输出如下

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530171739-spark-submit-output-demo-1.png)

### Columns and Expressions

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530173655-spark-py-sort-col-desc-demo.png)

Column objects in a DataFrame can’t exist in isolation; each column is part of a row in a record and all the rows together constitute a DataFrame, which as we will see later in the chapter is really a Dataset[Row] in Scala

### Rows

A row in Spark is a generic Row object, containing one or more columns. Each column may be of the same data type (e.g., integer or string), or they can have different types (integer, string, map, array, etc.). Because Row is an object in Spark and an ordered collection of fields, you can instantiate a Row in each of Spark’s supported languages and access its fields by an index starting at 0:

```shell
>>> from pyspark.sql import Row
>>> blog_row = Row(6, "Reynold", "Xin", "https://tinyurl.6", 255568, "3/2/2015",
...  ["twitter", "LinkedIn"])
>>> blog_row[1]
'Reynold'
```

create dataframes from row objects

```shell
>>> rows = [Row("Matei Zaharia", "CA"), Row("Reynold Xin", "CA")]
>>> authors_df = spark.createDataFrame(rows, ["Authors", "State"])
>>> authors_df.show()
+-------------+-----+
|      Authors|State|
+-------------+-----+
|Matei Zaharia|   CA|
|  Reynold Xin|   CA|
+-------------+-----+
```

### Common DataFrame Operations

了解一下DataFrameReader 和 DataFrameWriter

**Using DataFrameReader and DataFrameWriter**

```python
# Use the DataFrameReader interface to read a CSV file
sf_fire_file = "/databricks-datasets/learning-spark-v2/sf-fire/sf-fire-calls.csv"
fire_df = spark.read.csv(sf_fire_file, header=True, schema=fire_schema)
```

**Saving a DataFrame as a Parquet SQL table**

```python
# In Python to save as a Parquet file
parquet_path = ...
fire_df.write.format("parquet").save(parquet_path)

# In Python
parquet_table = ... # name of the table
fire_df.write.format("parquet").saveAsTable(parquet_table)
```

**Transformations and actions**

skip...

**Projections and filters**

投影和过滤

投影实际上是一种专业的用语， 比如使用filters 去匹配一些满足特定条件的Rows。

在Spark中，通过select（）去完成投影， filters 可以使用 filter() 或者 where()

**Renaming, adding, and dropping columns**  

```python
new_fire_df = fire_df.withColumnRenamed("Delay", "ResponseDelayedinMins")
(new_fire_df
 .select("ResponseDelayedinMins")
 .where(col("ResponseDelayedinMins") > 5)
 .show(5, False))


fire_ts_df = (new_fire_df
 .withColumn("IncidentDate", to_timestamp(col("CallDate"), "MM/dd/yyyy"))
 .drop("CallDate")
 .withColumn("OnWatchDate", to_timestamp(col("WatchDate"), "MM/dd/yyyy"))
 .drop("WatchDate")
 .withColumn("AvailableDtTS", to_timestamp(col("AvailableDtTm"),
 "MM/dd/yyyy hh:mm:ss a"))
 .drop("AvailableDtTm"))

# Select the converted columns
(fire_ts_df
 .select("IncidentDate", "OnWatchDate", "AvailableDtTS")
 .show(5, False))
```

**Aggregations**

```python
# In Python
(fire_ts_df
 .select("CallType")
 .where(col("CallType").isNotNull())
 .groupBy("CallType")
 .count()
 .orderBy("count", ascending=False)
 .show(n=10, truncate=False))
```

**Other common DataFrame operations**

```scala
// In Scala
import org.apache.spark.sql.{functions => F}
fireTsDF
 .select(F.sum("NumAlarms"), F.avg("ResponseDelayedinMins"),
 F.min("ResponseDelayedinMins"), F.max("ResponseDelayedinMins"))
 .show()
```

## Dataset API

 Datasets take on two characteristics: typed and untyped APIs,

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530183514-structured-apis-in-apache-spark.png)

Conceptually, you can think of a DataFrame in Scala as an alias for a collection of generic objects, Dataset[Row], where a Row is a generic untyped JVM object that may hold different types of fields. A Dataset, by contrast, is a collection of strongly typed JVM objects in Scala or a class in Java. Or, as the Dataset documentation puts it, a Dataset is: 

*a strongly typed collection of domain-specific objects that can be transformed in paral‐ lel using functional or relational operations. Each Dataset [in Scala] also has an unty‐ ped view called a DataFrame, which is a Dataset of Row*.

### Typed Objects, Untyped Objects, and Generic Rows

In Spark’s supported languages, Datasets make sense only in Java and Scala, whereas in Python and R only DataFrames make sense。

Python 和 R 不是编译类型语言， 不保证type-safe， types 是通过推断或者是执行之后指定的， 不是在编译的时候

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530183838-different-languages-datasets-dataframes.png)

Row is a generic object type in Spark, holding a collection of mixed types that can be accessed using an index. Internally, Spark manipulates Row objects, converting them to the equivalent types.

```shell
// In Scala
import org.apache.spark.sql.Row
val row = Row(350, true, "Learning Spark 2E", null)
# In Python
from pyspark.sql import Row
row = Row(350, True, "Learning Spark 2E", None)

// In Scala
row.getInt(0)
res23: Int = 350
row.getBoolean(1)
res24: Boolean = true
row.getString(2)
res25: String = Learning Spark 2E
# In Python
row[0]
Out[13]: 350
row[1]
Out[14]: True
row[2]
Out[15]: 'Learning Spark 2E'
```

### Creating Datasets

When creating a Dataset in Scala, the easiest way to specify the schema for the resulting Dataset is to use a case class. In Java, JavaBean classes are used

**Scala: Case classes**

比如json文件

```json
{"device_id": 198164, "device_name": "sensor-pad-198164owomcJZ", "ip":
"80.55.20.25", "cca2": "PL", "cca3": "POL", "cn": "Poland", "latitude":
53.080000, "longitude": 18.620000, "scale": "Celsius", "temp": 21,
"humidity": 65, "battery_level": 8, "c02_level": 1408,"lcd": "red",
"timestamp" :1458081226051}
```

使用case class

```scala
case class DeviceIoTData (battery_level: Long, c02_level: Long,
cca2: String, cca3: String, cn: String, device_id: Long,
device_name: String, humidity: Long, ip: String, latitude: Double,
lcd: String, longitude: Double, scale:String, temp: Long,
timestamp: Long)
```

```scala
// In Scala
val ds = spark.read
.json("/databricks-datasets/learning-spark-v2/iot-devices/iot_devices.json")
.as[DeviceIoTData]
ds: org.apache.spark.sql.Dataset[DeviceIoTData] = [battery_level...]
ds.show(5, false)
```

### Dataset Operations

```scala
val filterTempDS = ds.filter({d => {d.temp > 30 && d.humidity > 70})
filterTempDS: org.apache.spark.sql.Dataset[DeviceIoTData] = [battery_level...]
filterTempDS.show(5, false)
```

```scala
// In Scala
case class DeviceTempByCountry(temp: Long, device_name: String, device_id: Long,
 cca3: String)
val dsTemp = ds
 .filter(d => {d.temp > 25})
 .map(d => (d.temp, d.device_name, d.device_id, d.cca3))
 .toDF("temp", "device_name", "device_id", "cca3")
 .as[DeviceTempByCountry]
dsTemp.show(5, false)
```

To recap, the operations we can perform on Datasets—filter(), map(), groupBy(), select(), take(), etc.—are similar to the ones on DataFrames. In a way, Datasets are similar to RDDs in that they provide a similar interface to its aforementioned meth‐ ods and compile-time safety but with a much easier to read and an object-oriented programming interface. 

When we use Datasets, the underlying Spark SQL engine handles the creation, con‐version, serialization, and deserialization of the JVM objects. It also takes care of off-Java heap memory management with the help of Dataset encoders.

## DataFrames Versus Datasets

什么时候用 DataFrame or DataSet,  为什么这样选择

1. 如果你想告诉Spark需要做什么， 而不是如何去做， 使用DataFrame 或者 Dataset
2. 如果你想要丰富的语义， high-level 的抽象，DSL 操作， 使用DF 或者 DS
3. 如果你想要严格的编译类型正确， 并且不介意为特定的 Dataset[T] 创建多个case class 的话， 使用Dataset
4. 如果你的处理 要求high-level 的表达 filters， maps， aggregations (聚合)，计算平均数或者事求和， SQL 查询， 获取列， 或者是使用关系操作符， 在 semi-structured (半结构化)的数据上面， 请使用 DF 或者 DS
5. 如果您的处理要求进行类似于 SQL 查询的关系转换，使用DF
6. 如果你想得到Tungsten高效的编码序列化的优势的话， 使用DS
7. 如果你在Spark 组件之间想要使用统一，编码优化， 以及简洁的API的话， 使用DF
8. 如果你使用R， 用DF
9. 如果你使用Python， 使用DF， 如果你想要更多的控制的话， 把它丢到RDD
10. 如果你想要空间和速度的高效， 使用DF
11. 如果你想要编译时就能捕捉错误而不是在运行的时候， 参考下图

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220531101354-when-errors-are-detected-using-the-structured-apis.png)

### When to Use RDDs

RDD api 一直被保留， 2.x 和 3.x 都支持。但是 Spark 2.x 和 3.x 更偏向于是使用DF的接口和语义

但是也有一些场景你也可以考虑使用RDD

1. 比如有一些三方的包还是使用的 是RDD 去写的
2. 可以放弃DF，DS 带来的一些代码优化， 空间优化，和一些性能
3. 希望更简洁的让Spark 去做一些查询

实际上也可以使用

`df.rdd` 这个method call 去得到RDD， 但是这里面有一定的开销， 不建议使用。DF 和 DS 实际都是RDDs 的上层建筑，在whole-stage code generation 中， 他们被分解开来去适配RDD 代码。

## Spark SQL and the Underlying Engine

Spark SQL engine 如下：

1. 统一了Spark的组件， 允许多语言中的 DF/DS的抽象， 简化了结构化数据集的处理
2. 连接Apache Hive metastore 和 tables
3. 以特定的schema 从结构化的文件中， 读写结构话数据 （JSON，CSV等）， 并且将这些数据转化为临时表
4. 提供了一个Spark SQL Shell
5. 通过标准的JDBC/ODBC connectors, 构建了一个与外部工具之间的桥梁
6. Generates optimized query plans and compact code for the JVM, for final execution

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220531110031-spark-sql-and-its-stack.png)

### The Catalyst Optimizer

The Catalyst optimizer takes a computational query and converts it into an execution plan. It goes through four transformational phases。

1. Analysis 分析
2. Logical optimization 逻辑优化
3. Physical Planning 物理计划
4. Code generation 代码生成

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220531110312-spark-computation%E2%80%99s-four-phase-journey.png)

可以通过 explain 语句查询执行plan

```python
# In Python
count_mnm_df = (mnm_df
 .select("State", "Color", "Count")
 .groupBy("State", "Color")
 .agg(count("Count")
 .alias("Total"))
 .orderBy("Total", ascending=False))

count_mnm_df.explain(True)

-- In SQL
SELECT State, Color, Count, sum(Count) AS Total
FROM MNM_TABLE_NAME
GROUP BY State, Color, Count
ORDER BY Total DESC
```

```shell
== Parsed Logical Plan ==
'Sort ['Total DESC NULLS LAST], true
+- Aggregate [State#10, Color#11], [State#10, Color#11, count(Count#12) AS...]
 +- Project [State#10, Color#11, Count#12]
 +- Relation[State#10,Color#11,Count#12] csv
== Analyzed Logical Plan ==
State: string, Color: string, Total: bigint
Sort [Total#24L DESC NULLS LAST], true
+- Aggregate [State#10, Color#11], [State#10, Color#11, count(Count#12) AS...]
 +- Project [State#10, Color#11, Count#12]
 +- Relation[State#10,Color#11,Count#12] csv
== Optimized Logical Plan ==
Sort [Total#24L DESC NULLS LAST], true
+- Aggregate [State#10, Color#11], [State#10, Color#11, count(Count#12) AS...]
 +- Relation[State#10,Color#11,Count#12] csv
== Physical Plan ==
*(3) Sort [Total#24L DESC NULLS LAST], true, 0
+- Exchange rangepartitioning(Total#24L DESC NULLS LAST, 200)
 +- *(2) HashAggregate(keys=[State#10, Color#11], functions=[count(Count#12)],
output=[State#10, Color#11, Total#24L])
 +- Exchange hashpartitioning(State#10, Color#11, 200)
 +- *(1) HashAggregate(keys=[State#10, Color#11],
functions=[partial_count(Count#12)], output=[State#10, Color#11, count#29L])
 +- *(1) FileScan csv [State#10,Color#11,Count#12] Batched: false,
Format: CSV, Location:
InMemoryFileIndex[file:/Users/jules/gits/LearningSpark2.0/chapter2/py/src/...
dataset.csv], PartitionFilters: [], PushedFilters: [], ReadSchema:
struct<State:string,Color:string,Count:int>
```

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220531112842-an-example-of-a-specific-query-transformation.png)

**Phase 1: Analysis**
Spark SQL engine 为 SQL 或者是DF query 构建抽象语法树

**Phase 2: Logical optimization**

1. 首先会构建一组plans
2. 然后利用cost-based optimizer(CBO), 将 costs 列在每个plan上面

这些plans以操作树的形式构建出来，比如包括

常量折叠的处理， 预测下推，投影剪枝， 布尔表达式优化。Logical plan 是作为 physical 的输入的

**Phase 3: Physical planning**
这个阶段， Spark SQL 会利用Spark execution engine 中匹配的物理操作符， 从选定的logical plan 中创建优化过的physical plan

**Phase 4: Code generation**
最后一个阶段包含生成运行在每个机器上面的高效的Java 的字节码。因为Spark SQL 可以操作内存中的数据集，所以Spark 可以利用先进的code generation编译技术去加速执行。换句话说， 它表现的像一个编译器。Tungsten 项目就是干这件事的， 充当了一个whole-stage 的 code generation

什么是whole-stage code generation。 它是一个物理查询优化阶段， 它将整个查询， 分解为单个函数， 避免虚拟的函数调用， 并且利用CPU 寄存器存储中间数据。 第二代 Tungsten （introduced in Spark 2.0） 使用了这个技术去为最后的执行生成压缩的RDD代码。这种流线型的策略极大的改善了CPU的利用率和性能

