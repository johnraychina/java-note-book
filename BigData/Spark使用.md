http://spark.apache.org/docs/latest/index.html

# Spark是什么
通用大规模数据处理分析引擎。

提供各种语言的高级api: Java, Python, R

提供针对通用执行图的优化引擎

丰富的高级工具：
- 例如 Spark SQL结构化数据处理
- MLlib 机器学习
- GraphX 图处理
- Structured Streaming 增量计算和流式处理

## Spark 有什么能力，使用场景有哪些？

# Spark 安装使用
由于spark 使用了hadoop客户端库作为HDFS分布式文件存储 和 YARN调度
安装的时候你有两个选择：
1、选择预先打包了pre-packaged hadoop的spark版本；
2、选择无hadoop的安装包，然后在运行spark的时候再配置对hadoop的依赖；

Scala和Java用户可以在工程中通过maven坐标引入spark，
Python用户可以直接通过PyPI安装spark.


注意：Spark使用java运行，所以你必须先安装好java环境。
支持版本：java 8/11, Scala 2.12, Python 2.7+/3.4+, R 3.5+

spark下载后自带了示例，运行如下脚本（底层调用了spark-submit script来启动后台应用）：
java
```shell
./bin/run-example SparkPi 10
```

你也可以使用Scala shell来交互式地运行spark：
```shell
./bin/spark-shell --master local[2]
```
其中--master 指定了分布式集群的master URL
local[2]表示本机运行2个线程local[2]

spark还提供了Python shell来进行交互式地运行spark:
```shell
./bin/pyspark --master local[2]
```
当然也能提交后台运行：
```shell
./bin/spark-submit examples/src/main/python/pi.py 10
```

spark还提供了R shell：
```
./bin/sparkR --master local[2]
```
提交后台运行：
```shell
./bin/spark-submit examples/src/main/r/dataframe.R
```

# spark shell详细
./bin/spark-shell启动后，会提示可以进UI操作，还预备好了上下文、session。
Spark context Web UI available at http://30.38.68.56:4040
Spark context available as 'sc' (master = local[*], app id = local-1610766788133).
Spark session available as 'spark'.

# 启动集群
spark可以自己运行，也可以运行在集群管理器上。
- 单独部署模式
- Apache Mesos
- Hadoop YARN
- Kubernetes

# 核心组件
- RDD 
- SparkSQL， Dataset, DataFrames
- Structured Streaming
- Spark Streaming
- MLlib
- GraphX

## 弹性分布式数据集 RDD: resilient distributed dataset
file in Hadoop file sytem/Scala collection --> RDD


## 数据模型 Dataset
老版本spark2.0之前用RDD -> 最新版本用推荐Dataset，获得更佳性能。

## 共享变量shared variables:
- 广播变量broadcast variables：缓存
- 聚集变量accumulators(counter/sums)：追加

## Quick Start
安全选项：默认OFF关闭，你得自己手工打开。

**如何运行spark**
- 通过shell交互式使用spark数据分析: ./bin/spark-shell
- 通过提交后台运行spark数据分析: ./bin/spark-submit

**基本概念**
Dataset, DataFrame

[Dataset数据集API](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Dataset.html)
- 加载 spark.read.textFile("path");
- 过滤 filter
- 映射 map
- 聚合 reduce, count
- 短路 first
- 缓存：cache

**创建自包含应用** 
- Scala (with sbt)
    - sbt package 打包应用
    - 提交后台给spark运行
    YOUR_SPARK_HOME/bin/spark-submit \
    --class "SimpleApp" \
    --master local[4] \
    target/scala-2.12/simple-project_2.12-1.0.jar

- Java (with Maven)
    - maven package 打包应用
    - spark-submit 提交后台给spark运行
    YOUR_SPARK_HOME/bin/spark-submit \
    --class "SimpleApp" \
    --master local[4] \
    target/simple-project-1.0.jar

- Python (pip) 不用打包
方式1：直接spark-submit
方式2：先确保安装了pyspark(pip install pyspark) 
然后直接提交python解释器运行：
python SimpleApp.py

## 进一步学习：
- RDD 编程 [RDD programming guide](http://spark.apache.org/docs/latest/rdd-programming-guide.html)
- SQL 编程 [SQL programming guide](http://spark.apache.org/docs/latest/sql-programming-guide.html)
- 结构化流式处理 [Structured Streaming Programming guide](http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- 流式处理 [Spark Streaming](http://spark.apache.org/docs/latest/streaming-programming-guide.html)
- 机器学习 [MLlib for Machine Learning](http://spark.apache.org/docs/latest/ml-guide.html)
- 集群部署 [Cluster Mode](http://spark.apache.org/docs/latest/cluster-overview.html)


















