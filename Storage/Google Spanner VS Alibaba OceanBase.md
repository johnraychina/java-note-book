




## Spanner数据库是什么？
关系型数据库语义（ACID，事务，sql） + 全球级别的水平扩展（5个9的SLA）

一致性基于TrueTime + Paxos
TrueTime：GPS+原子钟 可以将不同数据中心的时间偏差缩短在10ms内，API可以提供一个精确时间+误差范围。
存储基于GFS(Clossus)

MegaStore: very close to Spanner

BigTable: 开源版本HBase


## OceanBase 

从架构上看，OceanBase 可以分为两层。

底层的分布式引擎和 Google Spanner 有很多共通之处：实现了线性可扩展、Paxos 复制、分布式事务、全局 schema 变更等等分布式特性；

上层的SQL引擎融合了传统关系数据库以及内存数据库的一些新技术。


Region 地域
Zone 可用区
RootService 总控服务
PartitionService 分区服务

OBServer 服务器节点
