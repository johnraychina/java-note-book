
# 基本原理

Google大数据三大论文： GFS, MapReduce, BigTable
GFS -> HDFS
MapReduce -> Hadoop -> Spark
BigTable -> HBase

# 架构图

# 核心设计

# RDD 编程
spark 应用：
由一个执行用户main函数的驱动程序，以及可以在集群上并行执行的并行操作组成。

什么是RDD：
spark 提供的主要抽象是 弹性数据集RDD，它是一组分区落在在集群节点中，并且可以被并行执行的数据集。
RDD 通常从Hadoop文件系统 或者 Scala 集合 转换而来。
用户也能将内存中的RDD 持久化，以便被并行操作复用。
RDD 能自动从节点宕机中恢复。

共享变量shared variables: 
- broadcast variables 广播变量
- accumulators(counters, sums): 聚合（只能add）




## 概要

## 链接 spark
应用引入依赖: spark-core  + hadoop client

## 初始化 spark
```scala
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
```

```java
SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
JavaSparkContext sc = new JavaSparkContext(conf);
```

```python
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
```

## 执行spark应用：shell或者提交后台
$ ./bin/spark-shell --master local[4] --jars code.jar
$ ./bin/spark-submit

## RDD 数据集
- parallelizing and existing collection
- referencing a dataset in an externel storage system(HDFS, HBase and any other Hadoop InputFormat.)

外部数据
scala> val distFile = sc.textFile(URI)

### RDD操作
- transformation 转换:
所有的转换采用惰性模式实现
可以通过persist/cache 持久化到内存中，也可以持久化到磁盘中，甚至复制到多个节点中。

map, flatMap, mapPartitions, filter, union, intersection, join, cartesian(笛卡尔全集)
distinct, 
groupByKey, cogroup,reduceByKey, aggregateByKey, sortByKey,
pipe,
coalesce
repartition
repartitionAndSortWithinPartitions


- actions 操作: 
lambda编程，将函数传入spark的操作中执行。
    - 理解闭包：函数不能改全局变量，只能 输入参数-> 函数 -> 输出结果
    - 使用K-V对

reduce, count, countByKey, first, foreach
take(n), takeSample, takeOrdered
saveAsTextFile
saveAsSequenceFile
saveAsObjectFile
collect



- shuffle 洗牌：有些操作需要将数据重新分派到不同分区（比如reduceByKey操作）
这非常消耗性能：磁盘IO，数据序列化+反序列化，网络IO


- RDD持久化 persist, cache, unpersist(spark自动按LRU淘汰缓存，也可以手工清理)
MEMORY_ONLY (default)
MEMORY_AND_DISK
MEMORY_ONLY_SER
MEMORY_AND_DISK_SER
DISK_ONLY
MEMORY_ONLY_2
MEMORY_AND_DISK_2
OFF_HEAP (experimental)


## 共享变量

https://www.infoq.cn/article/LBzKJPoaFAre5c0cI4ur
RDD 父子依赖关系 dependencies: NarrowDependency , ShuffleDependency

Spark 操作被洗牌（shuffle）切分为一个个的阶段（stage）。
Spark会自动将一个阶段（stage）内的同一份数据自动广播给阶段内的任务（task），这个数据会经过序列化和反序列化处理。
也就是说，在跨阶段（stage）或者需要缓存非序列化的数据时，显示地创建广播变量（broadcst variables）才有必要。

- broadcast variables 广播变量
广播: val broadCastVar = SparkContext.broadcast(v)
释放: broadCastVar.unpersist(), 如果后续还被使用，又会自动广播
干掉：broadCastVar.destroy(), 后续无法被使用

-  accumulators 聚集: count, sum
SparkContext.longAccumulator()
SparkContext.doubleAccumulator()

自定义聚集器：VectorAccumulatorV2 extends AccumulatorV2 
创建自定义聚集器 val myVectorAcc = new VectorAccumulatorV2
注册到上下文 sc.register(myVectorAcc, "MyVectorAcc1")


