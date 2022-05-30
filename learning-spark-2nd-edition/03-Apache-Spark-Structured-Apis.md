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

但是以上的设计会存在一些问题

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

A row in Spark is a generic Row object, containing one or more columns. Each col‐ umn may be of the same data type (e.g., integer or string), or they can have different types (integer, string, map, array, etc.). Because Row is an object in Spark and an ordered collection of fields, you can instantiate a Row in each of Spark’s supported lan‐ guages and access its fields by an index starting at 0:

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

When we use Datasets, the underlying Spark SQL engine handles the creation, con‐version, serialization, and deserialization of the JVM objects. It also takes care of offJava heap memory management with the help of Dataset encoders.