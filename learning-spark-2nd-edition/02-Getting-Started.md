# Downloading Apache Spark and Getting Started

## Downloading Spark

自己的电脑是 Mac M1， 可以直接使用homebrew 安装， homebrew 会将 spark shell, spark-submit 等等自动封装好， 然后在命令行里面可以直接敲出来

## Using Scala or PySpark Shell

skip...

## Understanding Spark Application Concepts

**Application**

A user program built on Spark using its APIs. It consists of a driver program and executors on the cluster

**Spark Session**
An object that provides a point of entry to interact with underlying Spark func‐ tionality and allows programming Spark with its APIs. In an interactive Spark shell, the Spark driver instantiates a SparkSession for you, while in a Spark application, you create a SparkSession object yourself.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530133642-spark-components-communicate-through-the-spark-driver-in-spark%E2%80%99s-distributed-architecture.png)

**Job**

A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action (e.g., save(), collect()).

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530134421-spark-driver-creating-one-or-more-spark-jobs.png)

**Stage**
Each job gets divided into smaller sets of tasks called stages that depend on each other.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530134707-spark-job-creating-one-or-more-stages.png)

**Task**

A single unit of work or execution that will be sent to a Spark executor

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530134849-spark-stage-creating-one-or-more-tasks-to-be-distributed-to-executors.png)

## Transformations, Actions, and Lazy Evaluation

Spark 的操作可以分为两种

1. transformations
2. actions

Transformations 顾名思义就是将 Spark的 DataFrame 转化为 新的 DataFrame， 这个过程中， 是不会改动原始数据的， 原始数据从而具有不可变的性质。 举个例子， 例如select() 或者 filter() 的操作， 是不会改变原始的DataFrame的，实际上以一个新的DataFrame 形式返回操作后的transformed results。

所有的transformation are evaluated lazily。

意思是，他们的计算结果不是立刻出来的， 而是以 一种流程线路 （血统，一脉相承）记录下来。这种记录下来的“流程”使得Spark，在其执行计划的稍后时间，重新安排某些转换，合并它们，或将转换优化为阶段以更有效地执行。这种Lazy evaluation 是Spark 延迟计算的一种策略， 直至action 被调用， 或者 数据检测到有读写

**lazy transformations and eager actions**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530142047-lazy-transformations-and-eager-actions.png)

Lazy evaluation 使得 Spark 可以通过窥探转化链来优化查询， 血统和数据的不可变性提供了容错。因为Spark 在血统中记录了每个转化， 而且转化之间DataFrame 是不变的， Spark 可以重新通过记录的血统， 通过回放重新构建原来的状态， 从而在容错中具有一点弹性

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530142754-transformations-and-actions-as-spark-operations.png)

```shell
# In Python
>>> strings = spark.read.text("../README.md")
>>> filtered = strings.filter(strings.value.contains("Spark"))
>>> filtered.count()
20
// In Scala
scala> import org.apache.spark.sql.functions._
scala> val strings = spark.read.text("../README.md")
scala> val filtered = strings.filter(col("value").contains("Spark"))
scala> filtered.count()
res5: Long = 20
```

### Narrow and Wide Transformations

Lazy evaluation 的好处：

Spark 可以审查你的计算查询，并且会判断一下如何优化。 这种优化可以通过joining或者pipelining 一些操作， 将他们指定分配个一个阶段， 或者是通过判定那些操作需要在集群上对数据进行shuffle 或者 exchange 将这些步骤拆解成为多个阶段。

**narrow dependencies**

一个数据分片可以由另外一个数据分片转化而来

比如filter()， contains() 类似

**wide dependencies**

比如groupBy() 和 orderBy() 等等操作， 可以认为是wide

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220530144257-narrow-versus-wide-transformations.png)

