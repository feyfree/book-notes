# Spark SQL and Datasets

Although we briefly introduced the Dataset API in Chapter 3, we skimmed over the salient aspects of how Datasets—strongly typed distributed collections—are created, stored, and serialized and deserialized in Spark.

## Single API for Java and Scala

Among the languages supported by Spark, only Scala and Java are strongly typed; hence, Python and R support only the untyped DataFrame API.

### Scala Case Classes and JavaBeans for Datasets

```scala
case class Bloggers(id:Int, first:String, last:String, url:String, date:String,
hits: Int, campaigns:Array[String])

val bloggers = "../data/bloggers.json"
val bloggersDS = spark
 .read
 .format("json")
 .option("path", bloggers)
 .load()
 .as[Bloggers]
```



```java
// In Java
import org.apache.spark.sql.Encoders;
import java.io.Serializable;
public class Bloggers implements Serializable {
 private int id;
 private String first;
  private String last;
 private String url;
 private String date;
 private int hits;
 private Array[String] campaigns;
// JavaBean getters and setters
int getID() { return id; }
void setID(int i) { id = i; }
String getFirst() { return first; }
void setFirst(String f) { first = f; }
String getLast() { return last; }
void setLast(String l) { last = l; }
String getURL() { return url; }
void setURL (String u) { url = u; }
String getDate() { return date; }
Void setDate(String d) { date = d; }
int getHits() { return hits; }
void setHits(int h) { hits = h; }
Array[String] getCampaigns() { return campaigns; }
void setCampaigns(Array[String] c) { campaigns = c; }
}
// Create Encoder
Encoder<Bloggers> BloggerEncoder = Encoders.bean(Bloggers.class);
String bloggers = "../bloggers.json"
Dataset<Bloggers>bloggersDS = spark
 .read
 .format("json")
 .option("path", bloggers)
 .load()
 .as(BloggerEncoder);
```

如果用在Scala 和 Java 中使用DS的话， 需要提前了解

1. column names and types
2. 和DF不一样， DF 可以通过Spark 推断数据类型， DS 则是要求你自己需要提前定义类型， 指定好schema

## Working with Datasets

One simple and dynamic way to create a sample Dataset is using a SparkSession instance. In this scenario, for illustration purposes, we dynamically create a Scala object with three fields: uid (unique ID for a user), uname (randomly generated user‐ name string), and usage (minutes of server or service usage).

### Creating Sample Data

```scala
// In Scala
import scala.util.Random._
// Our case class for the Dataset
case class Usage(uid:Int, uname:String, usage: Int)
val r = new scala.util.Random(42)
// Create 1000 instances of scala Usage class
// This generates data on the fly
val data = for (i <- 0 to 1000)
 yield (Usage(i, "user-" + r.alphanumeric.take(5).mkString(""),
 r.nextInt(1000)))
// Create a Dataset of Usage typed data
val dsUsage = spark.createDataset(data)
dsUsage.show(10)
```

```java
// In Java
import org.apache.spark.sql.Encoders;
import org.apache.commons.lang3.RandomStringUtils;
import java.io.Serializable;
import java.util.Random;
import java.util.ArrayList;
import java.util.List;
// Create a Java class as a Bean
public class Usage implements Serializable {
 int uid; // user id
 String uname; // username
 int usage; // usage
 public Usage(int uid, String uname, int usage) {
 this.uid = uid;
 this.uname = uname;
 this.usage = usage;
 }
 // JavaBean getters and setters
 public int getUid() { return this.uid; }
 public void setUid(int uid) { this.uid = uid; }
 public String getUname() { return this.uname; }
 public void setUname(String uname) { this.uname = uname; }
 public int getUsage() { return this.usage; }
 public void setUsage(int usage) { this.usage = usage; }
 public Usage() {
 }
 public String toString() {
 return "uid: '" + this.uid + "', uame: '" + this.uname + "',
 usage: '" + this.usage + "'";
 }
}
// Create an explicit Encoder
Encoder<Usage> usageEncoder = Encoders.bean(Usage.class);
Random rand = new Random();
rand.setSeed(42);
List<Usage> data = new ArrayList<Usage>()
// Create 1000 instances of Java Usage class
for (int i = 0; i < 1000; i++) {
 data.add(new Usage(i, "user" +
 RandomStringUtils.randomAlphanumeric(5),
 rand.nextInt(1000));

// Create a Dataset of Usage typed data
Dataset<Usage> dsUsage = spark.createDataset(data, usageEncoder);
```

scala 不需要写Encoder 的逻辑， 

### Transforming Sample Data

Scala is a functional programming language, and more recently lambdas, functional arguments, and closures have been added to Java too. Let’s try a couple of higherorder functions in Spark and use functional programming constructs with the sample data we created earlier.

**Higher-order functions and functional programming**

```scala
// In Scala
import org.apache.spark.sql.functions._
dsUsage
 .filter(d => d.usage > 900)
 .orderBy(desc("usage"))
 .show(5, false)

// In Scala
// Use an if-then-else lambda expression and compute a value
dsUsage.map(u => {if (u.usage > 750) u.usage * .15 else u.usage * .50 })
 .show(5, false)
// Define a function to compute the usage
def computeCostUsage(usage: Int): Double = {
 if (usage > 750) usage * 0.15 else usage * 0.50
}
// Use the function as an argument to map()
dsUsage.map(u => {computeCostUsage(u.usage)}).show(5, false)
```

**Converting DataFrames to Datasets**

```scala
// In Scala
val bloggersDS = spark
 .read
 .format("json")
 .option("path", "/data/bloggers/bloggers.json")
 .load()
 .as[Bloggers]
```

spark.read.format("json") returns a DataFrame, which in Scala is a type alias for Dataset[Row]. Using .as[Bloggers] instructs Spark to use encoders, discussed later in this chapter, to serialize/deserialize objects from Spark’s internal memory rep‐ resentation to JVM Bloggers objects.

## Memory Management for Datasets and DataFrames

Spark is an intensive in-memory distributed big data engine, so its efficient use of memory is crucial to its execution speed.1 Throughout its release history, Spark’s usage of memory has significantly evolved:

* Spark 1.0 使用了RDD， on the Java heap， 内存管理交给了JVM， JVM 来决定GC
* Spark 1.x 引入了 Tungsten ， off-heap 意味着 受GC的限制更小
  1. row-based format,  off-heap memory, using pointer and offset
  2. encoder 机制， 在JVM 和 Tungsten 格式序列化和反序列
* Spark 2.x 引入了 第二代的 Tungsten， 具备全阶段代码生成 和 向量形式的column-based 的 内存分布
* 采用现代编译器的理念， 更加重视当代CPU 和 缓存技术架构对于并行数据处理的理念（SIMD : single instruction, multiple data 单指令流多数据流 ）

## Dataset Encoders

Encoders convert data in off-heap memory from Spark’s internal Tungsten format to JVM Java objects. In other words, they serialize and deserialize Dataset objects from Spark’s internal format to JVM objects, including primitive data types. For example, an Encoder[T] will convert from Spark’s internal Tungsten format to Dataset[T].

Spark has built-in support for automatically generating encoders for primitive types (e.g., string, integer, long), Scala case classes, and JavaBeans. Compared to Java and Kryo serialization and deserialization, Spark encoders are significantly faster.

### Spark’s Internal Format Versus Java Object Format

Java objects have large overheads—header info, hashcode, Unicode info, etc. Even a simple Java string such as “abcd” takes 48 bytes of storage, instead of the 4 bytes you might expect. Imagine the overhead to create, for example, a MyClass(Int, String, String) object. 

Java 的对象空间占用很大， 空间利用率比较低

Instead of creating JVM-based objects for Datasets or DataFrames, Spark allocates off-heap Java memory to lay out their data and employs encoders to convert the data from in-memory representation to JVM object. For example, Figure 6-1 shows how the JVM object MyClass(Int, String, String) would be stored internally

### Serialization and Deserialization (SerDe)

为什么要用DS 的encoder

1. Spark 内部的 Tungsten 是 off-heap 存储， 而且压缩了数据
2. Encoder 可以快速的序列化出来这些数据， 在内存地址和偏移施加简单的指针算术
3. JVM 的 GC 停顿不影响 Encoder 的序列化和反序列化

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220606165005-spark-internel-tungsten-row-based-format.png)

### Costs of Using Datasets

JVM 和Tungsten 之间的形式转化是有开销的， 特别在数据量特别大的时候

### Strategies to Mitigate Costs

1. 不要大量使用匿名函数，能用DSL的， 就不要用匿名函数；Spark 对于匿名函数的优化， 可能不那么如意 Because lambdas are anonymous and opaque to the Catalyst optimizer until runtime, when you use them it cannot effi‐ ciently discern what you’re doing (you’re not telling Spark what to do) and thus can‐ not optimize your queries
2. The second strategy is to chain your queries together in such a way that serialization and deserialization is minimized. Chaining queries together is a common practice in Spark。 

```scala
import java.util.Calendar
val earliestYear = Calendar.getInstance.get(Calendar.YEAR) - 40
personDS
 // Everyone above 40: lambda-1
 .filter(x => x.birthDate.split("-")(0).toInt > earliestYear)

 // Everyone earning more than 80K
 .filter($"salary" > 80000)
// Last name starts with J: lambda-2
 .filter(x => x.lastName.startsWith("J"))

 // First name starts with D
 .filter($"firstName".startsWith("D"))
 .count()
```

each time we move **from lambda to DSL** (fil ter($"salary" > 8000)) we incur the cost of serializing and deserializing the Person JVM object.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220606172330-an-inefficient-way-to-chain-queries-with-lambdas-and-dsl.png)

```scala
personDS
 .filter(year($"birthDate") > earliestYear) // Everyone above 40
 .filter($"salary" > 80000) // Everyone earning more than 80K
 .filter($"lastName".startsWith("J")) // Last name starts with J
 .filter($"firstName".startsWith("D")) // First name starts with D
 .count()
```

尽量减少DSL 和 匿名函数的混用