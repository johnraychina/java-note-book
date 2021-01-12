http://spark.apache.org/docs/latest/index.html

# Spark是什么
通用大规模数据处理分析引擎。

提供各种语言的高级api: Java, Python, R

提供针对通用执行图的优化引擎

丰富的高级工具：
- 例如 Spark SQL结构化数据处理
- MLib 机器学习
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

# 启动集群



















