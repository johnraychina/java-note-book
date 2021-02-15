https://www.zhihu.com/question/19887265
https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/
http://leveldb.googlecode.com/svn/trunk/doc/impl.html
https://blog.csdn.net/high2011/article/details/83900147
https://cloud.tencent.com/developer/article/1441835

# 三种基本存储引擎

- Hash 存储引擎，是hash表的持久化实现。
支持随机读写，**不支持顺序扫描**，对应的存储系统为K-V存储系统。
对于K-V的插入和查询，复杂度都是O(1).

- B Tree（B-tree B+tree）存储引擎支持单条记录的增删改查，还支持顺序扫描（B+tree叶子节点之间的指针），对应的存储系统为MySQL等

- LSM Tree (Log-Structured Merge Tree) 同B树引擎一样，支持增删改查，顺序扫描。
而且通过批量存储技术来规避磁盘随机写入问题。
但是，和B+tree相比，LSM牺牲了部分读性能，用来大幅提高写性能。


LSM树的设计思想非常朴素：将对数据的修改保存在内存中，达到指定大小限制后，再将修改批量写入磁盘。
弊端是：读取的时候有点麻烦，需要合并磁盘中的历史数据和内存中最近修改的操作。
结果是：写入性能大大提升，读取时可能需要先看是否命中缓存，否则需要访问较多的磁盘文件。
极端的说，基于LSM树实现的HBase写入性能比MySQL高了一个数量级，读取性能低了一个数量级。


LSM树原理是：把一颗大树分为N棵小的**有序**的树，首先写入内存，
随着小树越来越大，内存中的小树会flush到磁盘中，
磁盘中的树定期做merge，合成一棵大树（按merge sort合成后仍然有序），以优化读取性能。

[HBase架构](HBase架构.jpg)
- 小树先写入内存，为了防止数据丢失，同时暂时持久化到磁盘，对应了HBase的MemStore和HLog；
- MemStore上的树达到一定大小之后，需要flush到HRegion磁盘中（一般是Hadoop DataNode)，
这样的MemStore就变成了DataNode上的磁盘文件StoreFile，
定期HRegionServer对DataNode上的数据做Merge操作，彻底删除无效空间，多棵小树在这个时候合并成大树，来增强读取性能。

LSM Tree，理论上，可以是内存树的一部分和磁盘中的第一层树做merge，对于磁盘中的树直接做update操作可能会破坏物理block的连续性，
但是在实际应用中，一般LSM有多层，当磁盘中的小树合并成一个大树的时候，可以重新排好序，使得block连续，优化读性能。

HBase在实现中，是把整个内存在达到一定阈值后，flush到disk，形成一个file，整个file的存储也就是一个小的B+树，
因为HBase一般部署在HDFS上，HDFS不支持对文件的update操作，所以HBase并非部分merge update，而是整体flush内存树。
flush到磁盘上的小树，定期会被合并到大树上。

整体而言，HBase就是用LSM树的思路。


# 深入理解 LSM-Tree

SSTable: Sorted String Table

[LSM-Tree架构](LSM-Tree架构.jpeg)
trade-off: 牺牲一点读取性能，换取很大的写入性能。
原理：内存树 + 整体顺序写磁盘 + 数据冗余 + 合并

- 内存数据(memtable)是可以更新的多叉树。
- 达到一定容量后，写入新的磁盘文件SSTable，变成只读的。
- SSTable compaction 周期性合并：如果有重复记录则用新的覆盖旧的数据。
- 查询时，首先按key查内存(memtable)，然后逆序逐个查sstable，
假设有k个sstable，每个N条记录，那么时间复杂度为 O(KlogN)

## 如何处理更新和删除呢？
更新：更新Memtable，然后后续的merge会覆盖旧的记录
删除：添加tombstone record “墓碑”记录


## 查询优化
查询性能虽然有所降低，但是可以做优化：
- 索引：每个sstable文件加索引
- 缓存：（也就是LevelDB的TableCache，将sstable索引缓存在LRU内存中）
- BloomFilter：如果bloom说一个key不存在于某个sstable，就一定不存在，从而减少查询文件数。
- SSTable可以开启压缩功能，而且并不是整个SSTable一起压缩，而是根据locality分组压缩，读取的时候不用解压整个文件，而只是解压部分group即可读取
- 合并：通过周期性合并文件，来减少sstable文件数。但Compaction操作是非常消耗CPU和磁盘IO的，尤其是在业务高峰期，如果发生了Major Compaction，则会降低整个系统的吞吐量，这也是一些NoSQL数据库，比如Hbase里面常常会禁用Major Compaction，并在凌晨业务低峰期进行合并的原因。

## SSTable Compaction 文件合并
为了保持LSM的读操作相对较快，维护并减少sstable文件的个数是很重要的，所以让我们更深入的看一下合并操作。
**这个过程有一点儿像一般垃圾回收算法。**

### Basic Compaction 基本合并
当一定数量的sstable文件被创建，例如有5个sstable，每一个有10行，他们被合并为一个50行的文件（或者更少的行数）。这个过程一 直持续着，当更多的有10行的sstable文件被创建，当产生5个文件时，它们就被合并到50行的文件。最终会有5个50行的文件，这时会将这5个50 行的文件合并成一个250行的文件。这个过程不停的创建更大的文件。像下图：

### Levelled compaction
上述的方案有个问题，就是大量的文件被创建，在最坏的情况下，所有的文件都要查询。
LevelDB 和 Cassandra 解决这个问题办法是：

1. 每一层可以维护指定的文件个数，同时保证不让key重叠。也就是说把key分区到不同的文件。因此在一层查找一个key，只用查找一个文件。
第一层是特殊情况，不满足上述条件，key可以分布在多个文件中。

2. 每次，文件只会被合并到上一层的一个文件。
当一层的文件数满足特定个数时，一个文件会被选出并合并到上一层。
这明显不同与另一种合并方式：一些相近大小的文件被合并为一个大文件。

## 总结
所以，LSM是日志和传统的单文件索引（B+tree，Hash Index）的中立，他提供一个机制来管理更小的独立索引文件（SSTable）

通过管理一组索引文件而不是单一的索引文件，LSM将B+数等昂贵的随机IO变得更快，而代价就是读取操作需要处理大量的索引文件（SSTable）而不是一个，另外Compaction还需要消耗一些IO。

## 关于 LSM 的一些思考

为什么 LSM 会比传统单个树结构有更好的性能？

我们看到LSM有更好的写性能，同时LSM还有其它一些好处。 sstable文件是不可修改的，这让对他们的锁操作非常简单。一般来说，唯一的竞争资源就是 memtable，相对来说需要相对复杂的锁机制来管理在不同的级别。所以最后的问题很可能是以写为导向的压力预期如何。如果你对LSM带来的写性能的提高很敏感，这将会很重要。大型互联网企业似乎很看中这个问题。 Yahoo 提出因为事件日志的增加和手机数据的增加，工作场景为从 read-heavy 到 read-write。
许多传统数据库产品似乎更青睐读优化文件结构。因为可用的内存的增加，通过操作系统提供的大文件缓存，读操作自然会被优化。

写性能（内存不可提高）因此变成了主要的关注点，所以采取其它的方法。
相对于写来说，硬件提升为读性能做的更多。因此选择一个写优化的文件结构很有意义。理所当然的，LSM的实现，像LevelDB和Cassandra提供了更好的写性能，相对于单树结构的策略。

## Beyond Levelled 

LSM这有更多的工作在LSM上， Yahoo开发了一个系统叫作 Pnuts， 组合了LSM与B树，提供了更好的性能。我没有看到这个算法的开放的实现。 IBM和Google也实现了这个算法。也有相关的策略通过相似的属性，但是是通过维护一个拱形的结构。如 Fractal Trees， Stratified Trees.

这当然是一个选择，数据库利用大量的配置，越来越多的数据库为不同的工作场景提供插件式引擎。 

Parquet存储格式

# SSTable
[SSTable](SSTable.jpeg)

# HBase 与 BigTable

