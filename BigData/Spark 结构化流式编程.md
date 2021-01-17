http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html

# 概览
结构化流式处理是一种可扩展，而且容错的流式处理引擎，它基于spark SQL引擎。
就像对静态数据做批处理一样，你能相同的方式表达对流式计算。

当数据源源不断地到来，spark SQL 引擎会增量、连续地更新最终结果。
你可以用Dataset/DataFrame API来表达流的聚集，事件窗口，流到批join等等。
这个计算执行在同一个优化的Spark SQL引擎上。
最后，系统通过检查点和追加日志(checkpoint+write ahead logs)提供 端到端 严格一次，容错保证能力。
总而言之，结构化流式处理提供快速的、可扩展的、容错的、端到端的、严格一次的流式处理能力，而无需用户来推测流的执行情况。

默认情况下，结构化流式查询在内部 是通过微批处理(micro-batch)引擎实现：
它将数据流当作一系列的小批处理作业，使得端到端延迟在低至100ms，而且实现【严格一致】的容错保证。

然而自从spark2.3起，我们引入了一种新的低延迟处理模式，叫做持续处理(Continuous Processing), 
它能在【至少一次】的容错保证下，达到端到端1ms的延迟。

在本篇指引中，我们将过一遍流式处理的编程模型和API。
我们将解释广泛使用的默认微批处理模型，然后再讨论持续处理模型(Continuous Processing)。

# 快速示例
写9999端口
$nc -lk 9999

监听9999端口
$./bin/run-example org.apache.spark.examples.sql.streaming.StructuredNetworkWordCount localhost 9999

# 编程模型
关键：将活动数据流当作一个被持续添加行数据的表。
这使得流式数据的处理模型与批处理模型非常类似。
你可以像对静态表一样来表达对流式数据的处理，spark使用无界输入表(unbounded input table)增量查询(incremental query)来实现。

## 基本概念
输入数据流，你可以看出“输入表”。
每次有数据到达，就像将行的行追加到“输入表”中。

对输入数据的查询，会产生“结果表”，每次触发期间（比如1秒钟），新的一行添加到“输入表”，最终触发对“结果表”的更新。
无论何时结果表被更新，我们都想将结果输出到外部的sink。

这里我们定义一下“输出”：被写到外部存储中的数据。输出可以被定义成不同的模式：
- Complete Mode 完整模式
- Append Mode 追加模式
- Update Mode 更新模式
注意：每种模式只能在[某些查询](http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#output-modes)当中使用.


## 处理 Event-time 和 Late Data


## 容错语义
如何提供端到端严格一次语义：
可重放的数据源（offset + checkpoint支持）
+ write-ahead-log
+ sink幂等

# 流式API与 Datasets, DataFrames


## 创建流式 DataFrames 和 流式 Datasets
SparkSession.readSream()

内置的几个输入源：
- 文件
- kafaka
- socket(仅用于测试) 没有提供端到端的容错保证，所以只能用于测试
- rate(仅用于测试) timestamp+value(Long)测试，基准测试。

一些数据源由于不能重放（不支持checkpoint+offset），所以不支持容错。


**流式DataFrames/Datasets的schema推断和分区**
默认情况下，基于文件的数据源，在结构化流式处理时，需要你指定schema，而非spark自动推断。
这个限制是为了确保一致的schema被用于流式查询，甚至是在失败处理时。
对一些特定的场景，你可以重新打开schema推断：spark.sql.streaming.schemaInference=true

当有/key=name/这种子目录出现的时候，会触发分区发现机制，而且自动递归。
如果这些列出现在用户提供的schema中，他们会被spark基于路径填充。

## 操作 流式 DataFrames 和 流式 Datasets
你可以对流式DataFrame/Datasets应用各种操作，包括
从无类型的，类SQL操作（select, where,groupBy等), 到有类型的 类RDD操作(map, filter, flatMap)等。
详细参见[SQL Programming Guide](http://spark.apache.org/docs/latest/sql-programming-guide.html)

### 基本操作 
selections, projection, aggregation

### Event Time窗口操作 
迟来的数据，要更新以前的窗口数据，那么我们保留多久以前的窗口状态呢？
Spark2.1引入水位机制(watermarking), 引擎自动跟踪当前数据的event time, 然后尝按阈值试清理过期的状态。

### join 操作 
- stream-static join 流-静态 join

```scala
val staticDf = spark.read...
val streamingDf = spark.readStream...
streamingDf.join(staticDf, "type") // inner equi-join with static DF
streamingDf.join(staticDf, "type", "right_join") //right out join with a static Df
```

#### stream-stream join 流-流 join

#### Inner Joins with optional Watermarking
面对新老数据join，老数据不可能一直保留，需要清理。
为了解决无界状态下，老数据清理问题，你需要额外定义join条件，让无限老的输入不能匹配将来的输入，这样才能让老数据安全清除。
换句话说，为了stream-stream join，你必须做如下额外工作：
1. 定义两个输入流的水位
2. 定义event-time约束，这样spark引擎就知道哪些老的数据不会被用于匹配其他输入。这个约束有两种方式：
    1. 时间范围join(例如 ...JOIN ON leftTime BETWEEN rightTime and rightTime + INTERVAL 1 HOUR),
    2. join event-time窗口(例如... JOIN ON leftTimeWindow = rightTimeWindow)

我们用一个例子来说明：
假设我们想join 广告影响impression(当广告出现的时候) 与 用户点击，用来计算什么时候 impression会产生收费点击。
为了对这个stream-stream join的状态及时清理，你需要指定水位延迟和时间约束：
1. 水位延迟Watermark delays: 比如impression 和 对应的点击量 可能对于event-time在2-3小时后过期。
2. Event-time范围条件：比如对应impression，一个点击可能出现再0~1小时内。

那么代码如下：
```scala
import org.apache.spark.sql.functions.expr

val impressions = spark.readStream. ...
val clicks = spark.readStream. ...

// Apply watermarks on event-time columns
val impressionsWithWatermark = impressions.withWatermark("impressionTime", "2 hours")
val clicksWithWatermark = clicks.withWatermark("clickTime", "3 hours")

// Join with event-time constraints
impressionsWithWatermark.join(
  clicksWithWatermark,
  expr("""
    clickAdId = impressionAdId AND
    clickTime >= impressionTime AND
    clickTime <= impressionTime + interval 1 hour
    """)
)
```
#### Outer Joins with Watermarking
watermark + event-time 约束
对于inner join来说是可选的，
但是对于left/right outer join则是必须的。


### 流去重
withWatermark
dropDuplicates

### 多watermarks处理策略
spark.sql.streaming.multipleWatermarkPolicy=min(default)

### 强制带状态的操作
mapGroupsWithState
flatMapGroupsWithState

### DataFrames/Datasets在流式类型下，不支持的操作
- 多重流嵌套聚合
- limit and take first N rows
- distinct
- 只支持 Complete Output Mode模式下聚合后的sort
- 部分流式Datasets的outer join

Dataset中一些立即返回的操作：
- count() - Cannot return a single count from a streaming Dataset. Instead, use ds.groupBy().count() which returns a streaming Dataset containing a running count.
- foreach() - Instead use ds.writeStream.foreach(...) (see next section).
- show() - Instead use the console sink (see next section).

### 全局watermark限制
- streaming aggregation in Append mode
- stream-stream outer join
- mapGroupsWithState and flatMapGroupsWithState in Append mode (depending on the implementation of the state function)



## 启动流式查询
Dataset.writeStream

- 指定输出 output sink: format(console/memory/parquet/orc/json/csv...), option(...)
- 输出模式 output mode: Append mode(default) / Complete mode / Update mode
- 查新名称 query name
- 触发间隔 trigger interval
- 检查点位置 checkpoint location

start(): 真正启动查询


## 管理流式查询
val query = df.writeStream.format("console").start()
```scala
val query = df.writeStream.format("console").start()   // get the query object

query.id          // get the unique identifier of the running query that persists across restarts from checkpoint data

query.runId       // get the unique id of this run of the query, which will be generated at every start/restart

query.name        // get the name of the auto-generated or user-specified name

query.explain()   // print detailed explanations of the query

query.stop()      // stop the query

query.awaitTermination()   // block until query is terminated, with stop() or with error

query.exception       // the exception if the query has been terminated with error

query.recentProgress  // an array of the most recent progress updates for this query

query.lastProgress    // the most recent progress update of this streaming query
```

## 监控流式查询

## 利用checkpoint 做失败恢复
```scala
aggDF
  .writeStream
  .outputMode("complete")
  .option("checkpointLocation", "path/to/HDFS/dir")
  .format("memory")
  .start()
```

## 恢复语义

# 持续处理(实验中)
指定continuous trigger: Trigger.Continuous

支持的查询：
- 操作：投影(select, map, flatMap, mapPartitions), 选择(where, filter)
- 数据源：kafka, rate
- 输出槽：kafaka, memory, console

## 小心一些坑
- 持续处理引擎会启动多个长期执行的task来读取、处理、写入数据，task数量取决于可以从数据源并行查询多少个partition，所以你要确保cpu 核数足够。
- 停止持续处理可能会产生一些奇怪的警告，你可以忽略他们。
- 目前对失败的task没有自动重试机制，你可以从checkpoint手工重启。


# 其他信息



