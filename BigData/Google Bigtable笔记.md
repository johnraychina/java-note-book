

## 2 数据模型
Rows: 对同一行的读写是原子的。

Row key: Bigtable按row key顺序组织数据。

Tablet: 表按行范围进行分区，每个行范围就是一个tablet，它是数据分布和负载均衡单位。
我们在访问数据的时候，可以充分利用这个locality特性来提高性能。

Column families: 列族由一系列相同类型的列组成，作为基本的访问单元、容量计算单元和权限控制单元。
比如一条网页记录，多国语言或者anchor都能作为column family。


Timestamps: 时间戳，多版本控制。
垃圾回收机制：指定版本数目 or 保留时间。

## 3 API
API支持对表和列族（column family)创建和删除操作。
API对单行的操作（read-modify-write)保证原子性，暂时不支持多 行事务，支持批量写，支持脚本。

## 4 Building Blocks 基础模块
-  集群管理系统：资源管理、任务调度等

- GFS 文件系统: 存储日志和数据文件

- SSTable 文件格式
持久化的有序的、不可修改的KV文件格式。
内部按block（大小可配置，默认是64KB）组织，block index被存储在SSTable末尾。
当打开SSTable文件时，把block index读入内存，先用二分搜索来定位block，然后再对block扫描，提升读取性能。

还有一招提升性能：通过mmap将SSTable整个映射到内存进行扫描。


- Chubby 分布式锁（对标zookeeper）
多副本，一主多从，paxos共识算法保持副本一致。
namespace 命名空间：一系列目录+文件，每个目录或者文件都能用来作为lock，对一个文件的读写都是原子的。
session：lease expire + renew 机制
callback：对目录或文件注册callback，当有修改或者session过期则发送通知。

用处：
BigTable服务器多副本，谁调Chubby加锁成功，谁就是master
发现Tablet服务器，对挂掉的Tablet服务器做清理。
存储BigTable的元数据（schema, 访问权限控制信息）


## 5 具体实现

客户端lib + One Master Server + Many Tablet Servers

 Master Server：
 - 分配tablet到tablet server
 - 管理tablet server：加机器、超时、负载均衡、GFS文件垃圾回收
 

Tablet server: 
- 管理tablet：（分裂）创建、读、写

和多数单master系统一样，客户端只是从master拿到tablet server信息，后续是直接与tablet server做数据交换。

### 5.1 Tablet 定位
Tablet METADATA元数据：tablet id， row key范围。

Tablet METADATA元数据文件三层树形结构，文件格式类似B+tree。
第一层元数据叫做root METADATA，存放在chubby文件中。
假设每条记录大概1KB，每个文件假设128MB，元数据文件三层树形结构，足以存储2^34条tablets元数据。
客户端可以缓存Tablet信息，这样可以最大程度减少master压力。


### 5.2 Tablet 分配

### 5.3 Tablet 服务

- 写入：校验权限，先写commit log， 然后写memtable -> SSTables
- 读取：读取METADATA + 读取合并SSTable+memtable


### 5.4 压缩
minor compaction
merge compaction
major compaction

## 6 完善
为了取得高性能、高可用性、可靠性，还需要做一些完善。

- locality groups 本地分组提升性能：相关性强的列，经常被一起访问，放在相同分组SSTable；相关性不强的列，分开存放不同SSTable，减少IO。
- compression 压缩：按block做压缩，读取的时候，可以by block解压读取。
- cache 缓存提高读性能： Scan Cache + Block Cache
- Bloom filters 布隆过滤器：没有对应key快速返回，避免无效的磁盘查找
- commit log 优化提交日志：
如果每个tablet单独写一个commit log，则需要对多个文件并发写入，这会导致对多个文件的seek（磁头移动）。
为了解决这个问题，每个tablet server所有tablets都写入一个物理文件log。
但这给失败恢复带来困难：一个tablet server挂掉后，需要利用commit log将它的tablets数据重新生成数据，均摊到其他的机器上。
怎么办呢：先对log排序即可，按{table, row name, log sequence number}排序（可以拆分为多个文件并行执行排序，然后merge），相同tablet的row会靠在一起。

- tablet恢复提速：先做一次压缩
If the master moves a tablet from one tablet server to another, the source tablet server first does a minor com- paction on that tablet. This compaction reduces recov- ery time by reducing the amount of uncompacted state in the tablet server’s commit log. After finishing this com- paction, the tablet server stops serving the tablet. Before it actually unloads the tablet, the tablet server does an- other (usually very fast) minor compaction to eliminate any remaining uncompacted state in the tablet server’s log that arrived while the first minor compaction was being performed. After this second minor compaction is complete, the tablet can be loaded on another tablet server without requiring any recovery of log entries.

- 最大限度利用不变性
    - 得益于SSTables的不变性，除了SSTable caches，还有BigTable系统各个其他的模块，都得到了极大简化：没有复杂的同步逻辑，并行处理变得非常高效，可变的数据只是memtable，对读写并发的场景利用copy-on-write技术。
    - 由于SSTable的不变性，物理删除数据的操作延后到了垃圾回收阶段。
    - 由于SSTable的不变性，tablets的分裂更加快速，我们不是每个子tablets生成一组新的SSTable，而是允许子tablets共享parent tablet里面的SSTables。

## 7 性能评估


## 8 实际项目使用情况
Google Analytics
Google Earth
Google Personalized Search

## 9 经验教训
- 一个教训是，我们发现，很多类型的错误都会导致大型分布式系统受损。
- 另外一个教训是，我们明白了在彻底了解一个新特性会被如何使用之后，再决定是否添加这个新特性是 非常重要的。比如事务处理。
- 还有一个具有实践意义的经验:我们发现系统级的监控对 Bigtable 非常重要。
- 对我们来说，最宝贵的经验是简单设计的价值。

