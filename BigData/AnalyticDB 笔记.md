
https://www.atatech.org/articles/111224
https://cloud.google.com/bigtable/docs/overview

Google，作为分布式大数据处理先行者，早在2003~2006年祭出三驾马车：GFS，BigTable，MapReduce。
将后辈 ADB 与 BigTable 两个数据仓库对比着看。
总体看下来，两者设计思想基本雷同。

ADB对标同行Teradata、Greenplum、Redshift、vertica等。

## 数据仓库要解决的问题：
数据量大，存储方面，单机无法支持：分而治之
数据量大，计算方面，单机查询慢：分而治之
高可用要求，单机无法支持：多副本
简化使用：简单的API，自动优化，自动资源管理和调度



## ADB实现亮点
### 1. 接入层
sql--> AST --> 逻辑执行计划operator --> 物理执行计划operator

解析：AnalyticDB 的SQL解析器采用阿里自研的高性能SQL Parser "fastsql"。

优化：利用元信息，优化SQL执行计划：
采用基于规则和基于代价的优化模式，支持分区减枝，CTE归并等SQL改写规则。
同时基于统计信息，实现最优JOIN ORDER等分布式查询路径的优化。

DAG计算模型自动优化参考[Spark AQE](https://docs.databricks.com/spark/latest/spark-sql/aqe.html)




### 2. 计算层：

读写分离：计算节点 + 写入节点，可隔离写入和分析计算负载。

计算节点利用本地SSD缓存数据。

引入 Code Generation 编译执行技术，极大较少CPU指令集，同时采用向量化运行，实现单实例的极致计算性能.
- [高效编译查询计划](Efficiently-Compiling-Efficient-Query-Plans.pdf)
- [hyper-db](http://www.hyper-db.com/)

引入GPU：将计算密集型操作卸载到GPU上，利用GPU的众核高并发能力，实现高性能计算。

AnalyticDB 采用Lambda架构，通过将历史数据和实时写入分开查询，并支持自动合并功能，提升整体查询性能和稳定性。




### 3. 存储层：

#### 元数据
丰富的Meta信息：分区、列、块 三个粒度的meta信息

#### 索引

#### 使用行列混合存储格式（chunk块存储）
按行切分：列是连续存储在一个地方，无需跨节点拼接多列，擅长OLTP明细查询。
按列切分：多行数据连续存储在一个地方，无需跨节点拼接多行，擅长OLAP大规模多维分析。

按行切分(row key)：类似的数据放一块，局部性原理。
按列切分(column type)：相同类型的列放一块，好压缩。

和BigTable的column family没两样，按行+列切分，是一个和稀泥的中庸做法。

定长块存储：解决负载不均问题（单个block过大，IO效率降低）。

及多种索引技术，支持将绝大部分计算条件基于索引完成。

分布式多副本 or [Erasure Code模式](https://en.wikipedia.org/wiki/Erasure_code)


#### 如何设置切分粒度
读写性能（处理时间）主要受这2个因素影响：
- data locality(尽量避免shuffle)
- workload balancing(尽量避免负载倾斜)

至于切分粒度与性能的关系，是一条凹曲线：
切分太细，则locality损伤严重，shuffle网络耗时高；
切分太粗，则workload不均衡，部分机器形成瓶颈；

```java
Object Function: Argument{切分粒度} to Min(总体处理时间) 
```


### 各种优化技术

#### 调度和负载均衡
组件：Operator, Driver, Pipeline, Task
嗯，看起来这和spark+beam有点像，又不完全对的号，桥豆麻袋，都没有详细介绍的吗？

避免快慢查询算子竞争CPU：
- 识别：基于统计数据，区分快慢查询算子。
- 解决：利用kernel sched，让执行快查询算子跑在高优先级线程中，慢查询算子跑在低优先级线程中。


#### CodeGen 代码生成
执行计划operator --> 解释为代码

早期采用 Volcano-style execution engine(火山执行引擎)

解释执行效率低：
- 需要查找函数表而非直接跳转目标代码
- cpu cache命中率低
- cpu 分支预测错误

使用ASM编译成代码 + 缓存




#### 硬件加速
1. GPU并行计算     
数据库中每条记录之间是独立的，非常适合GPU做并行计算，即使某些算子涉及多行记录之间的操作，例如Aggregation、order by、group by、hashjoin、set operation等，在算法层面也能做到算法并行。

例如Aggregation操作，N条记录的加法在CPU端实现，需要N-1次循环计算，时间周期是N-1，而在GPU中并行计算，只需要大概LOG(N)个加法周期。

同样的道理，hash join、group by、set operation（intersect、minus、union），distinct等操作的核心算法hash table，用CPU计算只能一条一条的计算，而使用GPU可以几千条数据同时计算，同时往hash表中插入，能几个数量级的提升计算的性能。


2. GPU代码生成：codegen+JCUDA动态生成GPU的执行代码
使用LLVM根据逻辑执行计划动态生成GPU物理执行计划。


3. GPU数据压缩/解压

参考文献

[1] N. Govindaraju,J. Gray, R. Kumar, and D. Manocha. Gputerasort: high performance graphics co-processorsorting for large database management. InSIGMOD, pages 325–336. ACM, 2006.
[2] B. He, K.Yang, R. Fang, M. Lu, N. Govindaraju, Q. Luo, and P. Sander. Relational joins ongraphics processors. In SIGMOD, pages 511–524, 2008.
[3] S.Yalamanchili. Scaling data warehousing applications using GPUs. In FastPath,2013.


#### 查询优化技术

1. 下推：对于存储-计算分离架构，通过下推计算提升性能；
2. 最优执行计划：基于代价选择最优的执行计划，动态规划算法；

#### 调度优化技术






