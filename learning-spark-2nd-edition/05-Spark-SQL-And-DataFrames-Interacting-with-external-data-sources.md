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



