http://spark.apache.org/docs/latest/streaming-programming-guide.html


- 概要
- 一个例子
- 基本概念
    - 链接
    - 初始化 StreamingContext
    - 离散型数据流 Discretized Stream(DStreams)
    - 输入DStream 与 接收 Reciever
    - DStream 转换
    - DStream 输出
    - DataFrame 与 SQL操作
    - 机器学习库的操作
    - 数据缓存和持久化
    - 检查点 Checkpoint
    - 累计、广播变量、检查点
    - 部署应用
    - 监控应用
- 性能调优
    - 减少批处理时间
    - 设置合适的批处理interval
    - 内存调优
- 容错语义


# 概要
spark streaming 是核心Spark API的扩展，
它能够横向扩展、高吞吐量、对在线流式数据提供容错的流式处理。

首先，输入数据源可以多种多样：Kafka, Kinesis, TCP sockets...
其次，可以用各种高级接口来表达复杂的处理逻辑：map, reduce, join, window...
最后数据还可以被push输出到文件系统、数据库、大屏幕看板。

实际上你还可以对数据流应用spark MLlib机器学习和 GraphX 图处理算法进行处理。

内部机理：(input data Stream)-->
spark streaming--(batches of inputdata) -->spark engine--(bathes of processed data)--> output storage

**DStream**
DStream(discretized stream):continuous stream of data.
内部机理：Dstream is represented as a sequence of RDDs.



# 一个例子
examples/src/main/scala/org/apache/spark/examples/streaming/NetworkWordCount.scala 
examples/src/main/java/org/apache/spark/examples/streaming/JavaNetworkWordCount.java

构建一个从socket接收网络字节流，split成word, 按一个batchDuration时间窗口reduceByKey统计每个word数量。
注意：
- 一个session只能启一个SparkContext，启动spark-shell默认就创建了sc = new SparkContext，
对于流式的context，你可以这么做:
```Scala
val ssc = new StreamingContext(sc, Seconds(10))
```

- ssc.start() 和  ssc.awaitTermination() 才算是启动scheduler开始计算DAG，前面都是构建DAG.

- 启动服务端
    - 可以走后台提交的方式： ./bin/run-example streaming.NetworkWordCount localhost 9999
    - 也可以直接启动spark-shell交互式输入服务端代码

- 新开一个shell，模拟客户端输入：
$ nc -lk 9999 
test
test
test



# 基本概念
## 链接 Linking
java开发引入jar包：
- spark-streaming 
- spark-streaming-kafka-0-10_2.12（像kafka这种数据源不在spark-streaming里面，需要额外引入对应的包）

## 初始化 StreamingContext
setAppName: 你的spark应用名，会显示在集群管理的界面上.
setMaster: 使用后台spark-submit提交集群的url(spark standalone, mesos, k8s, yarn ), 或者是 **"local[1]"** 本地模式跑

两种方式初始化流式上下文：
1. 利用已有的SparkContext： 
```Scala
val streamingContext = new StreamingContext(sparkContext, Seconds(1));
```
2. 新创建：
```Scala
val config = new SparkConfig().setAppName("appName").setMaster(url)
val streamingContext = new StreamingContext(config, Seconds(1))
```

streamingContext.start() //启动计算任务
streamingContext.awaitTermination() //等待计算结束
streamingContext.stop() //终止计算任务，默认会将整个上下文（流式+原始）全部关了（你可以修改参数），关掉就只能重新new，不能重启。

After a context is defined, you have to do the following.

1. Define the input sources by creating input DStreams.
2. Define the streaming computations by applying transformation and output operations to DStreams.
3. Start receiving data and processing it using streamingContext.start().
4. Wait for the processing to be stopped (manually or due to any error) using streamingContext.awaitTermination().
5. The processing can be manually stopped using streamingContext.stop().


**Points to remember:**
- Once a context has been started, no new streaming computations can be set up or added to it.
- Once a context has been stopped, it cannot be restarted.
- Only one StreamingContext can be active in a JVM at the same time.
- stop() on StreamingContext also stops the SparkContext. To stop only the StreamingContext, set the optional parameter of stop() called stopSparkContext to false.
- A SparkContext can be re-used to create multiple StreamingContexts, as long as the previous StreamingContext is stopped (without stopping the SparkContext) before the next StreamingContext is created.



## 离散型数据流 Discretized Stream(DStreams)
底层是一系列连续的RDD


### 输入 DStream 与 接收 Reciever
DStream 会从流式数据源接收数据流
DStream 会关联一个 Reciever 处理数据流
两种流式数据源：
- 基本数据源：StreamingContext API内置支持的文件系统，网络连接等
- 高级数据源：Kafka, Kinesis, etc. 这些要通过链接linking引入sdk

注意：本机跑spark的话，如果有开多个DStream并行处理，
设置master URL参数的时候， local[n], 必须n > Reciever个数，否则没线程接收网络数据了。


#### 文件流
DStream 可以从文件、 或者与HDFS API (that is, HDFS, S3, NFS, etc.)兼容的任何文件系统 读取数据：
StreamingContext.fileStream[KeyClass, ValueClass, InputFormatClass]
```Scala
streamingContext.textFileStream(dataDirectory)
```

**目录是监控的？**
spark streaming会监控处理 dataDirectory参数指定的目录：
- 按修改时间处理文件
- 简单目录
- POSIX glob pattern模式匹配
- 所有文件必须是相同的数据格式
- 当前时间窗口内，文件被处理后，不会重新处理（即使有更新）
- 一个目录下文件越多，扫描文件花费时间就越多-即使没有文件更改；
- 如果是用通配符指定目录，重新命名目录来匹配通配符后，目录会纳入监控；
- 调用 FileSystem.setTimes() 来修正时间戳，是一种让文件进入后续窗口中处理的办法（即使内容没变）。

#### 对象存储

**丢失文件更新问题**
对于“全”文件系统（比如HDFS），倾向在输出流被创建的时候，就设置修改时间。
所以当文件被打开，即使数据还没有写完，它可能被DStream纳入，
并且忽略同一个窗口内的文件更新，也就是说：可能会丢失文件更新，数据可能会被流忽略。

**解法：** 先写完文件，然后改名，使得文件纳入DStream目录。

相比之下，对象存储（比如Amazon S3和 Azure storge）的rename操作由于会做数据拷贝，通常会比较慢。
而且，重命名的对象可能会以rename的时间作为修改时间，而不是文件创建的时间，这里你要特别注意。
而且你要子系统测试，确保对象存储平台的时间与spark系统时间保持一致。

总之，你要仔细测试和推敲修改时间对你业务逻辑的影响。

#### 自定义 Reciver
[参见指引](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)

#### RDD 队列作为流
测试的时候你可以用RDD队列模拟流：
```Scala  
streamingContext.queueStream(queueOfRDDs)
```

#### 高级数据源
如果是后台运行spark应用，需要在你的应用中链接link引入依赖（kafka,kinesis等）。
如果你想在spark shell测试使用高级数据源，你需要将jar包放到classpath中.

### Reciever的可靠性
根据数据源可靠性分为不同数据源：是否支持ack机制（比如kafka）
可靠的Reciever：发送ack
不可靠的Reciever：数据源不支持ack 或者 不发ack

##  DStream 转换接口(类似RDD)
不会导致shuffle的：map, flatMap, filter, transform

可能导致shuffle的：repartition, union, count, reduce, countByValue, reduceByKey, join, cogroup

### updateStateByKey 状态更新操作
注意：使用updateStateByKey前需要设置checkpoint
他的机制是按key更新状态重算！！！
```Scala
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}

val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
```


### transform 转换操作
```Scala
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...)

val cleanedDStream = wordCounts.transform { rdd =>
    rdd.join(spamInfoRDD).filter(...)
}
```

### window 操作

window, countByWindow, reduceByWindow, reduceByKeyAndWindow,countByValueAndWindow

window length：窗口大小
sliding interval：滑动周期
```Scala
//reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow(a:Int, b:Int) =>
    (a+b), Seconds(30), Seconds(10))
```

### join 操作


**stream-stream  join**
```Scala
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```

```Scala
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream1.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2);
```

**stream-dataset join**
```Scala
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform{ rdd => rdd.join(dataset) }
```
实际上你可以动态修改你join的dataset，因为transform是在每个batch interval执行的，所以只会使用当前指向的dataset。





### DStream 输出
输出操作真正触发DStream的DAG计算（就像java stream的terminal操作触发intermediate操作）。

print, saveAsTextFiles, saveAsObjectFiles, saveAsHadoopFiles, 

foreachRDD：这个是最通用的，相当于把结果输出给你的function，你可以在function中自定义你的操作。

**使用foreachRDD的姿势（设计模式）**

错误姿势：在driver中创建网络连接
原因：spark 输入-计算-输出都是跑在worker机器上，driver只是发出命令。
```Scala
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

那么，我在函数里面开连接行不行？
跑是能跑，每条数据都开关一次连接，太慢了。
```Scala
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

正确的姿势：按分区处理（分区是spark分布式计算的最小调度单位）
这样我们的逻辑就能在分布式workers中高效跑起来了。
```Scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
```

最后，你还可以使用连接池的方式在多个RDD之间共享，更进一步减少建立网络连接的开销：
```Scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```


###  DataFrame 与 SQL操作

下面的例子，展示如何使用DataFrame做单词统计，
每个RDD被转换为DataFrame，接着注册为临时表，然后注册为临时表并用sql查询。

```Scala
val words: DStream[String] = ...

words.foreachRDD { rdd => 
    
    // Get the singleton instance of SparkSession
    val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()

    import spark.implicits._

    // Convert RDD[String] to DataFrame
    val wordsDataFrame = rdd.toDF("word")
    
    // Create a temporary view
    wordsDataFrame.createOrReplaceTempView("words")

    // Do word count on DataFrame using SQL and print it
    val wordsCountDataFrame = spark.sql("select  word, count(*) as total from words group by word")

    wordsCountDataFrame.show()
}
```

你也可以在不同线程中执行对流式数据的sql查询，
只是需要注意用streamingContext.remember 操作，
否则还没处理完这一批数据，下一批数据就更新过来了。







### 机器学习库的操作

### 数据缓存和持久化
基于窗口的操作（比如reduceByWindow, reduceByKeyAndWindow)
以及基于状态的操作（比如updateStateByKey）
不用自己调用persist(), spark默认就会把RDD persist到内存中。
对于从网络接收的数据流(kafka, sockets等)，默认的持久化等级是把数据复制到两个节点。


### 检查点 Checkpoint
示例代码：RecoverableNetworkWordCount.scala

和其他存储系统一样，spark checkpoint容错恢复用的。
有两种checkpoint数据类型：
- 元数据 Metadata checkpointing，包括配置、DStream操作、未完成的btach
- 数据 Data checkpointing

总的来说，元数据做checkpoint是为了driver的容错恢复，
而数据 或者 RDD 做checkpoint 是为worker计算的容错恢复。

#### 何时开启 checkpoint ?
以下场景是必须开启checkpoint的：
- 使用有状态的操作：比如updateStateByKey, reduceByKeyAndWindow(with invers function)
- 将运行应用的driver从失败中恢复

#### 如何配置 checkpoint ?
```Scala
streamingContext.checkpoint(checkpointDir)
```

如果你要应用从driver的失败中恢复，你还得让你的应用这么做：
- 应用第一次启动的时候，创建新的StreamingContext，设置所有的stream并调用 start()
- 如果应用是重失败中重启，你需要用checkpoint来重建StreamingContext


示例参见spark安装包中的例子：RecoverableNetworkWordCount.scala

```Scala
def functionToCreateContext(): StreamingContext =  {
    val ssc = new StreamingContext(...) //new context
    val lines = ssc.socketTextStream(...) //create DStreams
    ...
    ssc.checkpoint(checkpointDir) //set checkpoint directory
    ssc
}

// Get StreamingContext from checkpoint data or create new one
val context = StreamingContext.getOrCreate(checkpointDir, functionToCreateContext _)

// Do additional setup on context that needs to be done,
// irrespective of whether it is  being started or restarted.
context. ...

// Start the context
context.start()
context.awaitTermination()
```

checkpoint 保存状态依赖可靠存储（主要是hdfs）
就像redis，你需要平衡  失败恢复速度--运行时性能；


### 累计、广播变量、检查点
在spark streaming中，累计变量和广播变量不能从checkpoint恢复。
如果你开启了checkpoint，且使用累计变量和广播变量，你需要创建累计变量和广播变量的惰性单例，这样他们才能driver失败重启时恢复，示例如下：

```Scala
object WordBlacklist {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlacklist = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlacklist)
        }
      }
    }
    instance
  }
}

object DroppedWordsCounter {

  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("WordsInBlacklistCounter")
        }
      }
    }
    instance
  }
}

wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
  // Get or register the blacklist Broadcast
  val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
  // Get or register the droppedWordsCounter Accumulator
  val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
  // Use blacklist to drop words and use droppedWordsCounter to count them
  val counts = rdd.filter { case (word, count) =>
    if (blacklist.value.contains(word)) {
      droppedWordsCounter.add(count)
      false
    } else {
      true
    }
  }.collect().mkString("[", ", ", "]")
  val output = "Counts at time " + time + " " + counts
})
```


### 部署应用
- cluster with cluster manager
- package application jar
- configuring checkpointing
- configuring automatic restart of the application driver
    - spark standalone
    - yarn
    - mesos
- configuring write-ahead logs:
spark.streaming.receiver.writeAheadLog.enable

- setting the max receiving rate:
spark.streaming.receiver.maxRate
spark.streaming.backpressure.enabled: 通过backpressure自适应调整发送方速度


#### spark streaming应用升级
- 新老并行接收数据，新的ready后就把老的下掉
- 先把老应用优雅停机(StreamingContext.stop(...) or JavaStreamingContext.stop(...))，然后启动新的应用，从原来断点开始执行计算。注意：这种方式要求数据源支持停机期间对数据做buffer（比如kafka）。


### 监控应用
spark web UI监控界面，注意看两个指标：
- processing time: the time to process each batch of data
- scheduling delay - the time a batch waits in a queue for the processing of previous baths to finish.

你还可以通过API获取监控状态: StreamingListener

# 性能调优
总体上两种调优方法
1. 优化单个batch的处理效率
2. 每个设置合适的batch大小

## 优化单个批次处理效率
[详细参见调优指引](http://spark.apache.org/docs/latest/tuning.html)

## 接收数据的并行度调优

数据做拆分处理，然后再合并。
```Scala
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
```

调整receiver block interval: spark.streaming.blockInterval

每个batch的reciever的task数目大概 = (batch interval / block interval)
如果作业数目太少，你可以减小block interval，但是一般建议不超过50ms，太小的话task开销就会成问题。

另外一种提高数据接收并行度的方法是：对输入流repatition。

## 数据处理的并行度调优

如果某个阶段的并行计算task数不高，集群资源可能就没有被充分利用。
比如分布式的reduce操作: reduceByKey, reduceByKeyAndWindow, 并行任务数默认由 spark.default.parallelism 设置。你可以通过 DStream操作传参 或者 修改spark.default.parallelism 设置修改task并行度。


## 数据序列化调优
你可以通过调优序列化格式，来降低数据序列化的开销。
对于streaming流处理，有两种数据类型：
1. input data：从receiver接收的数据默认存储级别：StorageLevel.MEMORY_AND_DISK_SER_2.
2. streaming操作生成的持久化RDD，不同与spark core默认的StorageLevel.MEMORY_ONLY存储级别，流计算生成的持久化RDD默认是StorageLevel.MEMORY_ONLY_SER，以减小GC开销。

以上两种情况，使用Kryo序列化都能减小CPU和内存开销。对Kryo，考虑注册对应的class，并且禁用reference tracking.

## task启动开销调优

## 设置合适的batch批次大小
平衡延迟 -- 吞吐量
## 内存调优
关于spark应用的内存使用和GC行为，强烈建议参考[调优指引](http://spark.apache.org/docs/latest/tuning.html#memory-tuning).

spark streaming 应用内存相关的参数调优：

spark streaming 需要的内存与你的操作息息相关：
map-filter-store就不怎么占内存
window操作需要你的集群有足够内存来存储窗口期内的数据
updateStateByKey 如果操作的key非常多，那么就需要非常大的内存。

spark streaming 接收的数据时，按默认的存储级别StorageLevel.MEMORY_AND_DISK_SER_2, 如果内存不够则会把数据存到硬盘。

另外一个你要考虑的就是调优GC，尤其是对低延迟有要求的应用。

有几个参数可以用于调优内存 和 GC 开销：
- 调整DStream的持久化等级，使用Kryo，数据压缩（cpu换内存）
- 清除旧数据，DStream的操作默认会自动清理，但有时候你如果设置了 streamingContext.remember，就需要自己及时清理。
- GC算法目前推荐用CMS，GC停顿会小一点。
- 其他：使用OFF_HEAP级别持久化RDD， 将机器资源拆分为更多的executor和更小的heap，这样GC压力会小一点。

## 重点注意
- 一个DStream只关联一个reciever，要并行读取，得创建多个DStream，一个reciver运行在一个executor中，占用一个cpu核心，你要确保有足够的核心来处理其他的计算。

- 当spark从流数据源接收数据时，reciever会创建数据块，每个interval创建一个数据块，数据块个数N= batchInterval/blockInterval, 这些数据块会被当前executor的BlockManager发到其他的BlockManger上，然后被其他executor处理，然后driver上的Network Input Tracker会被通知block的位置，便于后续处理。

- 在一个batchInterval批次周期内的blocks会对应在driver上创建一个RDD，而这些blocks就是RDD分区。每个分区是spark上的一个任务。blockInterval==batchInterval意味着只有一个分区，可以本地处理。

- 越大的blockInterval意味着越大的blocks，越大的 spark.locality.wait 意味着更大的开了在本节点处理。你需要平衡好这两个参数，尽量本地处理。

- 除了依赖batchInterval和blockInterval，你还可以通过inputDStrea.reparition(n)修改分区数，这会把数据随机分不到n个分区。更大的并行度 和 shuffle的代价需要你自己权衡。

- 如果你有2个DStream，就会对应2套RDD， 分别跑在2个job中，你可以union DStream减少job。

- 如果batch处理时间比batchInterval长，那reciver内存没被及时释放，新的数据又进来，最后可能导致溢出（主要是BlockNotFoundException）。目前没有办法停掉receiver，但是你可以用 spark.streaming.receiver.maxRate限速.

# 容错语义

1. Data received and replicated
2. Data received but buffered for replication

1. Failure of a Worker Node
2. Failure of the Driver Node 
 

流处理系统的三种可靠性保障：
1. 最多一次
2. 最少一次
3. 严格一次

流处理三个基本步骤：
1. 接收数据：不同数据源提供不同保障
2. 转换数据：严格一次
3. 输出数据：默认是至少一次，用户自己做幂等处理

