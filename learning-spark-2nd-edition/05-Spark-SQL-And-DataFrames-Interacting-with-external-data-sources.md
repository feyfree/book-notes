# Spark SQL and DataFrames: Interacting with External Data Sources

理解Spark SQL 带来的功能

1. 对于Apache Hive 和 Apache Spark 可以使用自定义函数
2. 连接外部数据源： JDBC， SQL 数据库， Tableau，Azure Cosmos DB等等
3. 和简单还有复杂的类型， 一些高级函数和普通的关系运算符打交道

## Spark SQL and Apache Hive

Spark SQL 是一个Spark 的基础组件， 用Spark 函数式编程的API， 集成了关系数据处理。

一开始起源叫做Shark， Shark 一开始是在 Hive 的代码库上面开发的， 这个是在Spark 上面的应用， 后来成为Hadoop 系统里面比较早的交互SQL查询引擎。当时号称是两全其美的

1. 像商业的数据仓库那么快
2. 像Hive/MapReduce 那么容易伸缩扩容 （scaling）

Spark SQL 允许 Spark 程序员利用更快的性能和关系型编程 （比如声明查询和优化存储）， 以及调用复杂的分析库（比如机器学习）

### User-Defined Functions

尽管Apache Spark 有很多内置的函数， Spark 也允许开发人员编写他们自己的函数： user-defined-functions (UDFS)

**Spark SQL UDFs**

可以自定义函数

```python
# In Python
from pyspark.sql.types import LongType
# Create cubed function
def cubed(s):
 return s * s * s
# Register UDF
spark.udf.register("cubed", cubed, LongType())
# Generate temporary view
spark.range(1, 9).createOrReplaceTempView("udf_test")
// In Scala/Python
// Query the cubed UDF
spark.sql("SELECT id, cubed(id) AS id_cubed FROM udf_test").show()
+---+--------+
| id|id_cubed|
+---+--------+
| 1| 1|
| 2| 8|
| 3| 27|
| 4| 64|
| 5| 125|
| 6| 216|
| 7| 343|
| 8| 512|
+---+--------+
```

**Evaluation order and null checking in Spark SQL**

Spark SQL (包括SQL， DF， DS 的api)不会保证 evaluation 的顺序

比如

```shell
spark.sql("SELECT s FROM test1 WHERE s IS NOT NULL AND strlen(s) > 1")
```

where 后面的条件顺序并不能保证在 evaluation 的时候， 谁在前， 谁在后

所以对于null 的检查， 推荐

1. UDF 需要去识别这些null， 并且在UDF 中进行null 检查
2. 使用 IF 或者 CASE WHEN 语句去做null check， 在一个条件分支中去调用UDF

**Speeding up and distributing PySpark UDFs with Pandas UDFs**

PySpark 的UDFs 比 Scala 慢？

因为Pyspark UDF 会有数据的移动，从JVM上面， 这种开销还是蛮大的。

Pandas UDFs 优化了这个问题 （Apache 2.3）， 使用了 Apache Arrow format， 不再需要去序列化这些数据， 因为这些数据已经是Python 进程可以识别的。 以往处理单个输入都是 row by row， 现在你可以直接操作在Pandas Series 或者是DF 上面

1. Pandas UDFs
2. Pandas Function APIs

Spark 3.0 后， Pandas UDFs 会从Python 类型中进行推断， 比如 pandas.Series, pandas.DataFrame, Tuple and Iterator. 首先你需要手动定义和制定Pandas UDF type,  然后这些在Pandas UDFs 中支持的Python 类型 会和Scala 中的有对应的匹配

Pandas Function APIs 允许你只直接应用本地的python 方法到PySpark 的DF 上面，  这些DF 也许既作为Pandas 的输入和输入。 在Spark 3.0，支持的Pandas Function APIs 比如grouped map， map， cogrouped map

```python
# In Python
# Import pandas
import pandas as pd
# Import various pyspark SQL functions including pandas_udf
from pyspark.sql.functions import col, pandas_udf
from pyspark.sql.types import LongType
# Declare the cubed function
def cubed(a: pd.Series) -> pd.Series:
 return a * a * a
# Create the pandas UDF for the cubed function
cubed_udf = pandas_udf(cubed, returnType=LongType())
# Create a Pandas Series
x = pd.Series([1, 2, 3])
# The function for a pandas_udf executed with local Pandas data
print(cubed(x))

The output is as follows:
0 1
1 8
2 27
dtype: int64
```

Pandas 的一些示例这边不展开了

## Querying with the Spark SQL Shell, Beeline, and Tableau

### Spark SQL Shell

```shell
spark-sql> CREATE TABLE people (name STRING, age int);
22/06/01 17:18:37 WARN ResolveSessionCatalog: A Hive serde table will be created as there is no table provider specified. You can set spark.sql.legacy.createHiveTableByDefault to false so that native data source table will be created instead.
22/06/01 17:18:38 WARN SessionState: METASTORE_FILTER_HOOK will be ignored, since hive.security.authorization.manager is set to instance of HiveAuthorizerFactory.
22/06/01 17:18:38 WARN HiveConf: HiveConf of name hive.internal.ss.authz.settings.applied.marker does not exist
22/06/01 17:18:38 WARN HiveConf: HiveConf of name hive.stats.jdbc.timeout does not exist
22/06/01 17:18:38 WARN HiveConf: HiveConf of name hive.stats.retries.wait does not exist
22/06/01 17:18:38 WARN HiveMetaStore: Location: file:/Users/feyfree/spark-warehouse/people specified for non-external table:people
Time taken: 1.555 seconds

spark-sql> INSERT INTO people VALUES ("Michael", NULL);
Time taken: 0.285 seconds

spark-sql> show tables;
people
Time taken: 0.067 seconds, Fetched 1 row(s)
spark-sql> select * from percent
percent_rank(        percentile(          percentile_approx(   
spark-sql> select * from people where age < 20;
Samantha	19
Time taken: 0.279 seconds, Fetched 1 row(s)
```

### Working with Beeline

Beeline 实际上就是JDBC 客户端， 基于SQL的命令行。

首先创建一个Thrift JDBC/ODBC 的服务器， 这个服务器是连接到 HiveServer上面的。

通过beeline 脚本可以连接到Spark 或者是 hive

**start the Thrift server**
To start the Spark Thrift JDBC/ODBC server, execute the following command from the $SPARK_HOME folder:

```shell
./sbin/start-thriftserver.sh
```

**Connect to the Thrift server via Beeline**

```shell
./bin/beeline

➜  ~ spark-beeline 
log4j:WARN No appenders could be found for logger (org.apache.hadoop.util.Shell).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
Beeline version 2.3.9 by Apache Hive
beeline> show tables;
No current connection
beeline> !connect jdbc:hive2://localhost:10000
Connecting to jdbc:hive2://localhost:10000
Enter username for jdbc:hive2://localhost:10000: 
Enter password for jdbc:hive2://localhost:10000: 
Could not open connection to the HS2 server. Please check the server URI and if the URI is correct, then ask the administrator to check the server status.
Error: Could not open client transport with JDBC Uri: jdbc:hive2://localhost:10000: java.net.ConnectException: Connection refused (Connection refused) (state=08S01,code=0)
beeline> 
```

**Stop the Thrift server**

```shell
./sbin/stop-thriftserver.sh
```

### Working with Tableau

略

## External Data Sources

通过Spark SQL 连接到其他数据源

### JDBC and SQL Databases

```shell
./bin/spark-shell --driver-class-path $database.jar --jars $database.jar
```

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220601180046-spark-shell-connection-properties.png)

**Partitioning的重要性**

数据量可能很大， 一个连接传输的数据如果太大的话， 性能和稳定性都有风险， 对于大型数据的操作的话， 推荐使用下面的一些参数

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220601181046-partitioning-connection-props.png)

需要注意几点

1. A good starting point for numPartitions is to use a multiple of the number of Spark workers。 分区是worker 的整数倍， 尽量减少数据库的并发请求 （比如OLTP）, 可能会导致负载过饱
2. 提前熟悉数据分布， 制定合理的 lowerBound 和 upperBound,  避免一些无效的查询， 比如数据分布是 200 到 400， 如果设置的 上下界是 0 - 1000， 可能spark 的查询是 0 - 100 查一次， 100 - 200 查一次这样子， 那样会有一些无效的查询
2. Choose a partitionColumn that can be uniformly distributed to avoid data skew. For example, if the majority of your partitionColumn has the value 2500, with {numPartitions:10, lowerBound: 1000, upperBound: 10000} most of the work will be performed by the task requesting the values between 2000 and 3000. Instead, choose a different partitionColumn, or if possible generate a new one (perhaps a hash of multiple columns) to more evenly distribute your partitions. 翻译过来就是避免数据倾斜， 如果partitionColumn 都是value = 2500， 那么查询肯定都会在 2000 - 3000 这个请求里面， 尽量使得数据查询能分摊一下， 避免数据倾斜。

### 一些数据库的连接方式

略。。。

## Higher-Order Functions in DF and Spark SQL

对于复杂的数据类型， 两种典型的处理方式

1. 嵌套的数据结构拆解为独立的行， 应用特定的函数， 然后重建这个嵌套数据结构
2. 编写用户自定义函数

### Option 1: Explode and Collect

```sql
-- In SQL
SELECT id, collect_list(value + 1) AS values
FROM (SELECT id, EXPLODE(values) AS value
 FROM table) x
GROUP BY id
```

While collect_list() returns a list of objects with duplicates, the GROUP BY state‐ ment requires shuffle operations, meaning the order of the re-collected array isn’t necessarily the same as that of the original array. As values could be any number of dimensions (a really wide and/or really long array) and we’re doing a GROUP BY, this approach could be very expensive

### Option 2: User-Defined Function

```shell
spark.sql("SELECT id, plusOneInt(values) AS values FROM table").show()
```

While this is better than using explode() and collect_list() as there won’t be any ordering issues, the serialization and deserialization process itself may be expensive. It’s also important to note, however, that collect_list() may cause executors to experience out-of-memory issues for large data sets, whereas using UDFs would alle‐ viate these issues

### Built-in Functions for Complex Data Types

略

### Higher-Order Functions

`transform()`

`filter()`

`exists()`

`reduce()`

## Common DataFrames and Spark SQL Operations

* Aggregate functions 

* Collection functions

* Datetime functions

*  Math functions 

* Miscellaneous functions 

* Non-aggregate functions 

* Sorting functions 

* String functions

* UDF functions

* Window functions

### Unions Joins Windowing (Windowing 略) Modification 

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import expr

if __name__ == "__main__":
    spark = (SparkSession
             .builder
             .appName("chapter5-practice")
             .getOrCreate())
    tripdelaysFilePath = "../databricks-datasets/learning-spark-v2/flights/departuredelays.csv"
    airportsnaFilePath = "../databricks-datasets/learning-spark-v2/flights/airport-codes-na.txt"

    # Obtain airports data set
    airportsna = (spark.read
                  .format("csv")
                  .options(header="true", inferSchema="true", sep="\t")
                  .load(airportsnaFilePath))

    airportsna.createOrReplaceTempView("airports_na")

    # Obtain departure delays data set
    departureDelays = (spark.read
                       .format("csv")
                       .options(header="true")
                       .load(tripdelaysFilePath))
    departureDelays = (departureDelays
                       .withColumn("delay", expr("CAST(delay as INT) as delay"))
                       .withColumn("distance", expr("CAST(distance as INT) as distance")))
    departureDelays.createOrReplaceTempView("departureDelays")
    # Create temporary small table
    foo = (departureDelays
           .filter(expr("""origin == 'SEA' and destination == 'SFO' and
     date like '01010%' and delay > 0""")))
    foo.createOrReplaceTempView("foo")

    spark.sql("SELECT * FROM airports_na LIMIT 10").show()
    spark.sql("SELECT * FROM departureDelays LIMIT 10").show()
    spark.sql("SELECT * FROM foo").show()

    # Union 操作
    print("Here is Union Demo -----")
    bar = departureDelays.union(foo)
    bar.createOrReplaceTempView("bar")
    # Show the union (filtering for SEA and SFO in a specific time range)
    bar.filter(expr("""origin == 'SEA' AND destination == 'SFO'
    AND date LIKE '01010%' AND delay > 0""")).show()
    print("Here is Union Demo End -----")

    # Join 操作
    # Join departure delays data (foo) with airport info
    print("Here is Join Demo -----")
    foo.join(
        airportsna,
        airportsna.IATA == foo.origin
    ).select("City", "State", "date", "delay", "distance", "destination").show()
    print("Here is Join Demo End-----")

    # Windows 略

    # Modification
    # add new columns
    foo2 = (foo.withColumn(
        "status",
        expr("CASE WHEN delay <= 10 THEN 'On-time' ELSE 'Delayed' END")
    ))
    foo2.show()

    # dropping columns
    foo3 = foo2.drop("delay")
    foo3.show()

    # renaming
    foo4 = foo3.withColumnRenamed("status", "flight_status")
    foo4.show()

    # pivoting 略

```

**windowing function**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220606153118-spark-sql-window-functions.png)