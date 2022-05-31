# Spark SQL and DataFrames: Introduction to Built-in Data Sources

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220531110031-spark-sql-and-its-stack.png)

## Using Spark SQL in Spark Applications

You can use a SparkSession to access Spark functionality: just import the class and create an instance in your code

### Basic Query Examples

```shell
>>> from pyspark.sql import SparkSession
>>> spark=(SparkSession.builder.appName("SparkSQLExampleApp").getOrCreate())
>>> csv_file="/Users/feyfree/Develop/OpenSource/LearningSparkV2/databricks-datasets/learning-spark-v2/flights/departuredelays.csv"
>>> df=(spark.read.format("csv").option("inferSchema", "true").option("header", "true").load(csv_file))
>>> df.createOrReplaceTempView("us_delay_flights_tbl")
>>> spark.sql("""SELECT distance, origin, destination
... FROM us_delay_flights_tbl WHERE distance > 1000
... ORDER BY distance DESC""").show(10)
+--------+------+-----------+
|distance|origin|destination|
+--------+------+-----------+
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
+--------+------+-----------+
only showing top 10 rows
>>> (df.select("distance", "origin", "destination")
...  .where("distance > 1000")
...  .orderBy("distance", ascending=False).show(10))
+--------+------+-----------+
|distance|origin|destination|
+--------+------+-----------+
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
|    4330|   HNL|        JFK|
+--------+------+-----------+
only showing top 10 rows
```

对于关系数据库表的查询， 手写SQL 和用 Spark SQL interface 效果是差不多的

To enable you to query structured data as shown in the preceding examples, Spark manages all the complexities of creating and managing views and tables, both in memory and on disk. That leads us to our next topic: how tables and views are cre‐ ated and managed.

## SQL Tables and Views

Tables 存储数据。 Spark 中的每个table 有相关的元数据，是关于table 还有数据的一些信息

比如schema， 描述， 表名， 数据库名，列表名，分区， 物理地址 （数据实际存储的地方）等等， 这些都是存储在一个central metastore

Spark 默认使用Apache Hive metastore (located at `/user/hive/warehouse`), 去存储所有的表的metadata。当然你可以改默认的地址

### Managed Versus UnmanagedTables

Spark 允许创建两种tables

1. managed
2. unmanaged

managed， Spark 不仅管理metadata还有data in the file store. 可能是本地的文件，HDFS， 或者是Amazon S3 的对象存储， 或者是Azure

unmanaged, Spark 只管理metadata，你自己需要管理外部的数据， 比如存储在Cassandra 上面的数据

managed table, 因为Spark 管理一切， 所以一个 SQL 比如 `DROP TABLE table_name`这个操作既会删除元数据， 也会删除表的数据， 但是 对于unmanaged table， Spark 只会删除 元数据， 真实数据不会删除

### Creating SQL Databases and Tables

```scala
// In Scala/Python
spark.sql("CREATE DATABASE learn_spark_db")
spark.sql("USE learn_spark_db")
```

**Creating a managed table**

```scala
// In Scala/Python
spark.sql("CREATE TABLE managed_us_delay_flights_tbl (date STRING, delay INT,
 distance INT, origin STRING, destination STRING)")

# In Python
# Path to our US flight delays CSV file
csv_file = "/databricks-datasets/learning-spark-v2/flights/departuredelays.csv"
# Schema as defined in the preceding example
schema="date STRING, delay INT, distance INT, origin STRING, destination STRING"
flights_df = spark.read.csv(csv_file, schema=schema)
flights_df.write.saveAsTable("managed_us_delay_flights_tbl")
```

**Creating an unmanaged table**

```scala
spark.sql("""CREATE TABLE us_delay_flights_tbl(date STRING, delay INT,
 distance INT, origin STRING, destination STRING)
 USING csv OPTIONS (PATH
 '/databricks-datasets/learning-spark-v2/flights/departuredelays.csv')""")

// DF API
(flights_df
 .write
 .option("path", "/tmp/data/us_flights_delay")
 .saveAsTable("us_delay_flights_tbl"))
```

### Creating Views

创建视图

1. 全局
2. session-scoped

视图不保存数据，tables 是持久化的， 而view 会随着 Spark 应用终止而消失

```sql
-- In SQL
CREATE OR REPLACE GLOBAL TEMP VIEW us_origin_airport_SFO_global_tmp_view AS
 SELECT date, delay, origin, destination from us_delay_flights_tbl WHERE
 origin = 'SFO';
CREATE OR REPLACE TEMP VIEW us_origin_airport_JFK_tmp_view AS
 SELECT date, delay, origin, destination from us_delay_flights_tbl WHERE
 origin = 'JFK'
```

```python
# In Python
df_sfo = spark.sql("SELECT date, delay, origin, destination FROM
 us_delay_flights_tbl WHERE origin = 'SFO'")
df_jfk = spark.sql("SELECT date, delay, origin, destination FROM
 us_delay_flights_tbl WHERE origin = 'JFK'")
                   
# Create a temporary and global temporary view
df_sfo.createOrReplaceGlobalTempView("us_origin_airport_SFO_global_tmp_view")
df_jfk.createOrReplaceTempView("us_origin_airport_JFK_tmp_view")
```

Keep in mind that when accessing a global temporary view you must use the prefix **global_temp**., because Spark creates global temporary views in a global temporary database called **global_temp**

```sql
-- In SQL
SELECT * FROM global_temp.us_origin_airport_SFO_global_tmp_view
```

**normal temporary view** 

```sql
-- In SQL
SELECT * FROM us_origin_airport_JFK_tmp_view
// In Scala/Python
spark.read.table("us_origin_airport_JFK_tmp_view")
// Or
spark.sql("SELECT * FROM us_origin_airport_JFK_tmp_view")
```

**drop a view**

```sql
-- In SQL
DROP VIEW IF EXISTS us_origin_airport_SFO_global_tmp_view;
DROP VIEW IF EXISTS us_origin_airport_JFK_tmp_view
```

```scala
// In Scala/Python
spark.catalog.dropGlobalTempView("us_origin_airport_SFO_global_tmp_view")
spark.catalog.dropTempView("us_origin_airport_JFK_tmp_view")
```

**Temporary views versus global temporary views**

a temporary view 绑定到某个spark 应用中的单个 SparkSession

a global tempory view 是全局的， 单个spark 应用中所有的SparkSession 都可以用 （比如某些session 并没有共享相同的 hive metastore 配置）

### Viewing the Metadata

```python
// In Scala/Python
spark.catalog.listDatabases()
spark.catalog.listTables()
spark.catalog.listColumns("us_delay_flights_tbl")
```

### Caching SQL Tables

Although we will discuss table caching strategies in the next chapter, it’s worth men‐ tioning here that, like DataFrames, you can cache and uncache SQL tables and views. In Spark 3.0, in addition to other options, you can specify a table as LAZY, meaning that it should only be cached when it is first used instead of immediately:

```sql
-- In SQL
CACHE [LAZY] TABLE <table-name>
UNCACHE TABLE <table-name>
```

### Reading Tables into DataFrames

```scala
// In Scala
val usFlightsDF = spark.sql("SELECT * FROM us_delay_flights_tbl")
val usFlightsDF2 = spark.table("us_delay_flights_tbl")

# In Python
us_flights_df = spark.sql("SELECT * FROM us_delay_flights_tbl")
us_flights_df2 = spark.table("us_delay_flights_tbl")
```

## Data Sources for DataFrames and SQL Tables

It also provides a set of common methods for reading and writing data to and from these data sources using the Data Sources API

比如Spark 提供了多种访问各种数据源的interfaces， 它也提供了这些访问后， 数据读写的一些API

### DataFrameReader

```scala
// In Scala
// Use Parquet
val file = """/databricks-datasets/learning-spark-v2/flights/summary-
 data/parquet/2010-summary.parquet"""
val df = spark.read.format("parquet").load(file)
// Use Parquet; you can omit format("parquet") if you wish as it's the default
val df2 = spark.read.load(file)
// Use CSV
val df3 = spark.read.format("csv")
 .option("inferSchema", "true")
 .option("header", "true")
 .option("mode", "PERMISSIVE")
 .load("/databricks-datasets/learning-spark-v2/flights/summary-data/csv/*")
// Use JSON
val df4 = spark.read.format("json")
 .load("/databricks-datasets/learning-spark-v2/flights/summary-data/json/*")
```

### DataFrameWriter

```scala
// In Scala
// Use JSON
val location = ...
df.write.format("json").mode("overwrite").save(location)
```

### Some Formats

Spark 同时提供了一些比如 JSON， CSV， Parquet 等等的一些读写转化， 可以搜索一下

