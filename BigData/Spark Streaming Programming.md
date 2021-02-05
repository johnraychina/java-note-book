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

updateStateByKey
注意：使用updateStateByKey前需要设置checkpoint
他的机制是按key更新状态重算！！！


- DStream 输出
- DataFrame 与 SQL操作
- 机器学习库的操作
- 数据缓存和持久化
- 检查点 Checkpoint
- 累计、广播变量、检查点
- 部署应用
- 监控应用


# 性能调优

# 容错语义


 