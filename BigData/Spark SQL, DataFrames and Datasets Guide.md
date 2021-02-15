http://spark.apache.org/docs/latest/sql-programming-guide.html

DataFrame in Java: Dataset<Row>
DataFrame in Scala: Dataset[Row]


# 动手开始
1. 交互式使用scala命令行
```shell
./bin/spark-shell
```

2. 交互式执行sql
```shell
./bin/spark-sql
```

3. 交互式使用python:
```shell
./bin/pyspark
```

4. 提交后台执行脚本 or jar包
```shell
./bin/spark-submit
```

```Scala
// 初始化session
val spark = SparkSession.builder()
  .appName("test")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()

// 创建DataFrame
val df = spark.read.json("examples/src/main/resources/people.json")
```

## 无类型的Dataset操作（即DataFrame操作）
http://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Dataset.html
DataFrame为各种编程语言提供DSL操作结构化的数据操作，可以用 Scala, Java, Python and R.
如前所述，spark 2.0中，DataFrame只是Dataset的行 DataSet[Row].
相对于Java/Scala中Dataset的“强类型操作”，这些操作也叫做“无类型操作”。
[RDD-DataFrame-Dataset对比](https://databricks.com/blog/2016/07/14/a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets.html)
[对比图](Spark-SQL-Dataframes-Datasets.png)

```scala
df.printSchema()
df.select("name").show()
df.select($"name", $"age" + 1).show()
df.filter($"age" > 21).show()
df.groupBy("age").count().show()
```

## SQL查询
```scala
df.createOrReplaceTempView("people")
val sqlDF = spark.sql("SELECT * FROM people")
sqlDF.show()
```

## 全局临时视图(需要配置hive存储)
普通的临时视图，生命周期是session范围的，
想要创建所有session都能看到的临时视图，就得创建全局的createGlobalTempView,
这种视图在spark应用停止前都存在。

df.createGlobalTempView("people")
spark.sql("SELECT * FROM global_temp.people").show()
spark.newSession().sql("SELECT * FROM global_temp.people").show()

## 创建DataSet
"examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala"
Dataset与RDD类似，但不是用Java自带序列化或者Kryo，
而是用特殊的Encoder 来序列化和传输对象。
虽然encoder和标准的序列化会将对象转换为bytes,
但是encoder是动态生成的，
而且encorder会使用一种格式，**这种格式允许spark不做反序列化直接执行许多操作**：比如filter, sort, hash等。




## 与RDD互操作
- 方式一：通过RDD+map Person反射推断schema
val peopleDF = spark.sparkContext.
  textFile("examples/src/main/resources/people.txt").
  map(_.split(",")).
  map(attributes => Person(attributes(0), attributes(1).trim.toInt)).
  toDF()

- 方式二：编程式
- Create an RDD of Rows from the original RDD;
- Create the schema represented by a StructType matching the structure of Rows in the RDD created in Step 1.
- Apply the schema to the RDD of Rows via createDataFrame method provided by SparkSession.



## 标量函数scalar functions
http://spark.apache.org/docs/latest/sql-ref-functions.html#scalar-functions
http://spark.apache.org/docs/latest/sql-ref-functions-udf-scalar.html

## 聚合函数aggregate functions

# 数据源
示例代码："examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 

## load/save 通用存取函数, 支持option
```scala
val df = spark.read.format("csv")
  .option("sep", ";")
  .option("inferSchema", "true")
  .option("header", "true")
  .load("examples/src/main/resources/people.csv")

usersDF.write.format("orc")
  .option("orc.bloom.filter.columns", "favorite_color")
  .option("orc.dictionary.key.threshold", "1.0")
  .option("orc.column.encoding.direct", "name")
  .save("users_with_options.orc")
```


### 直接对文件执行sql
```scala
val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")
```

### SaveMode 保存模式
SaveMode.ErrorIfExists
SaveMode.Append
SaveMode.Overwrite
SaveMode.Ignore

### saveAsTable 保存到持久化表中
df.write.saveAsTable("t")
保存到默认的warehouse directory路径中，并且将每个分区元数据存到hive metastore_db元数据存储中。

df.write.option("path", "/some/path").saveAsTable("t")
保存到指定的路径中。
但这种方式不会将metadata存到hive元数据存储中，需要你手工同步到hive：MSCK REPAIR TABLE.

读取出来
spark.table("people").show()

分桶、排序、分区:
分桶、排序只能应用于持久化表: bucketedBy, sortBy ==> saveAsTable
分区 partitionBy ==> save和saveAsTable

### 通用文件选项
基于文件的数据源：parquet, orc, avro, json, csv, text

通用选项：spark.sql.files
- 忽略受损的文件 ignoreCorruptFiles
- 忽略丢失的文件 ignoreMissingFiles
- 路径全局过滤 pathGlobFilter
- 文件递归查找 recursiveFileLookup （与 partitionSpec选项互斥）


### Parquet文件格式
Partition Discovery:
spark.sql.sources.partitionColumnTypeInference.enabled=true(default)
path/to/table/gender=male

Schema Merging: mergeSchema

### Hive metastore parquet table 的转换
写非分区的Hive metastore Parquet table时，有2种方式：
默认用自己的Parquet support，性能好一些;
还有就是Hive SerDe;
spark.sql.hive.convertMetastoreParquet=true

#### Hive/Parquet schema 协调
Hive是大小写敏感，Parquet不是
Hive认为所有列都是可空的，而while nullability in Parquet is significant

#### 元数据刷新
为了提升性能，SparkSQL 会缓存Parquet的元数据，为了保证元数据的一致性，你可以手工刷新：
spark.catalog.refreshTable("my_table")


编程式，设置选项：spark.setConf("key", value)
SQL，设置选项: SET key=value 

### ORC Files
spark.sql.orc.impl=native
spark.sql.orc.enableVectorizedReader=true

### Hive Tables

# Spark SQL 性能调优
[性能调优](http://spark.apache.org/docs/latest/sql-performance-tuning.html)
缓存数据
优化配置（有序可能由于自动优化而下掉一些配置）
join hints 关联查询相关hint
coalesce hints 合并分区
spark AQE自适应查询优化：
- 合并shuffle后的分区
- 将sort-merge join转换为boradcast join
- 优化数据倾斜的join

## Caching Data In Memory
spark.catalog.cacheTable("tableName") or dataFrame.cache()
spark.catalog.uncacheTable("tableName")

configuration:
spark.sql.inMemoryColumnarStorage.compressed
spark.sql.inMemoryColumnarStorage.batchSize	

## Other Configuration Options
spark.sql.files.maxPartitionBytes
spark.sql.files.openCostInBytes
spark.sql.broadcastTimeout
spark.sql.autoBroadcastJoinThreshold
spark.sql.shuffle.partitions
spark.sql.sources.parallelPartitionDiscovery.threshold
spark.sql.sources.parallelPartitionDiscovery.parallelism	



## Join Strategy Hints for SQL Queries
join hints的优先级 
BORADCAST > MERGE > SHUFFLE_HASH > SHUFFLE_REPLICATE_NL


## Coalesce Hints for SQL Queries
coalesce hint 让你可以控制输出文件的个数，就像 Dataset API中的 coalesce, repartition, repartitionByRange 操作。

```SQL
SELECT /*+ COALESCE(3) */ * FROM t
SELECT /*+ REPARTITION(3) */ * FROM t
SELECT /*+ REPARTITION(c) */ * FROM t
SELECT /*+ REPARTITION(3, c) */ * FROM t
SELECT /*+ REPARTITION_BY_RANGE(c) */ * FROM t
SELECT /*+ REPARTITION_BY_RANGE(3, c) */ * FROM t
```
[coalesce VS repartition](https://stackoverflow.com/questions/31610971/spark-repartition-vs-coalesce)
这两位回答最好：Justin Pihony + Powers
coalesce: 合并分区。
repartition: 重新分区，数据重新分布。


AQE(Adaptive Query Execution)
```Java
spark.sql.adaptive.enabled=true(default false)
```

As of spark 3.0, there are Three major features:
```Java
// 1. Coalescing Post Shuffle Partitions
spark.sql.adaptive.coalescePartitions.enabled
spark.sql.adaptive.coalescePartitions.minPartitionNum	
spark.sql.adaptive.coalescePartitions.initialPartitionNum
spark.sql.adaptive.advisoryPartitionSizeInBytes	
// 2. Converting sort-merge join to broadcast join
spark.sql.adaptive.localShuffleReader.enabled

// 3. Optimizing Skew Join
spark.sql.adaptive.skewJoin.enabled
spark.sql.adaptive.skewJoin.skewedPartitionFactor
spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes
```

# 分布式sql引擎 Distributed SQL Engine
## Running the Thrift JDBC/ODBC server
```shell
# To start the JDBC/ODBC server, run the following in the Spark directory:
./sbin/start-thriftserver.sh
./sbin/start-thriftserver.sh --help
```

```shell
# Now you can use beeline to test the Thrift JDBC/ODBC server
./bin/beeline

# Connect to the JDBC/ODBC server in beeline with:
beeline> !connect jdbc:hive2://localhost:10000
```

## Running the Spark SQL CLI
```shell
./bin/spark-sql
./bin/spark-sql --help
```
Configuration of Hive is done by placing your hive-site.xml, core-site.xml and hdfs-site.xml files in conf/. 




# PySaprk + Pandas + Apache Arrow 指引
http://spark.apache.org/docs/latest/sql-pyspark-pandas-with-arrow.html
示例详情参见spark安装目录中"examples/src/main/python/sql/arrow.py"

Apache Arrow: 内存基于列的数据格式，JVM<-->Python进程传输数据
基于列、相同的数据类型，适合cpu的SIMD向量化处理，加速计算。

```shell
pip install pyspark
./bin/pyspark
```

```Python
import numpy as np
import pandas as pd

# Enable Arrow-based columnar data transfers
spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", "true")

# Generate a Pandas DataFrame
pdf = pd.DataFrame(np.random.rand(100, 3))

# Create a Spark DataFrame from a Pandas DataFrame using Arrow
df = spark.createDataFrame(pdf)

# Convert the Spark DataFrame back to a Pandas DataFrame using Arrow
result_pdf = df.select("*").toPandas()
```

## Pandas UDFs (a.k.a Vectorized UDFs)

**pandas_udf**
A Pandas UDF is defined using the pandas_udf as a decorator or to wrap the function, and no additional configuration is required.

## Series to Series
The type hint can be expressed as pandas.Series, … -> pandas.Series.

输入输出长度一样，可以按列拆分为多个批次，然后每个批次作为数据集调用函数，然后将结果连接起来。
Internally, PySpark will execute a Pandas UDF by splitting columns into batches and calling the function for each batch as a subset of the data, then concatenating the results together.

## Iterator of Series to Iterator of Series
The type hint can be expressed as Iterator[pandas.Series] -> Iterator[pandas.Series].


## Iterator of Multiple Series to Iterator of Series
The type hint can be expressed as Iterator[Tuple[pandas.Series, ...]] -> Iterator[pandas.Series].
## Series to Scalar
The type hint can be expressed as pandas.Series, … -> Any.

# SQL 手册
需要注意的CREATE TABLE的一些参数：


**USING data_source**

Data Source is the input format used to create the table. Data source can be CSV, TXT, ORC, JDBC, PARQUET, etc.

**PARTITIONED BY**

Partitions are created on the table, based on the columns specified.

**CLUSTERED BY**

Partitions created on the table will be bucketed into fixed buckets based on the column specified for bucketing.

NOTE: Bucketing is an optimization technique that uses buckets (and bucketing columns) to determine data partitioning and avoid data shuffle.

**SORTED BY**

Determines the order in which the data is stored in buckets. Default is Ascending order.

