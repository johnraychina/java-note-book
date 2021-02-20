


编程模型：map-reduce 简单，通用，能很好表达各种场景的计算逻辑。

框架：帮你做资源调度、并行计算、负载均衡、网络通信、容错处理等。

实例：词频统计，分布式grep，词向量，反向链接，反向索引（word-doc list），分布式排序

实现：

执行：split-map-reduce
如何将计算逻辑分发到worker机器上执行？
Map和Reduce的数据输入、输出怎么定义和存储？


Master数据结构：

容错处理：worker宕机重试，master宕机重试

