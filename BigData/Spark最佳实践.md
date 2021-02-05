
[最佳实践01](Spark最佳实践01.pdf)
1、如何加速 文件加载与分区发现

即使是spark懒加载机制，
为了查询计划，spark需要知道一些元数据（schema，文件列表、分区）和统计信息（文件大小）等
所以会启动job做一些工作(schema inference, partition discovery, list files, file size, etc)

方案1：直接load指定文件具体文件而不是load数据源
If you know exactly what partitions you want:
- If you need to use files vs Datasource Tables
- Specify “basePath” option and load specific partition paths 
- InMemoryFileIndex “pre-filters” on those paths

方案2：使用spark.read.table + filter,而不是load数据源+filter
- Use Datasource Tables with Spark 2.1+
- Partitions managed in Hive metastore
- Partition Pruning at logical planning stage
- Scalable Partition Handling for Cloud-Native Architecture in Apach 2.1
 

读取CSV和JSON文件：
spark会对**整个**数据集做schema推断，不慢才怪。
Avoiding Schema Inference
- Easiest - use Tables!
- Otherwise...If consistent data, infer on subset of files 
- Save schema for reuse (e.g. scheduled batch jobs)
- JSON - adjust samplingRatio (default 1.0)



数据源表因为有更加充分的元数据，所以加载性能比文件要好：
Datasource Tables vs Files
- Managed, more scalable partition handling
- Schema in metastore
- Faster job startup
- Additional statistics available to Catalyst (e.g. CBO)
- Use SQL or DataFrame API

- Partition discovery at each DataFrame creation
- Infer schema from files 
- Slower job startup time
- Only file-size statistics available
- Only DataFrame API (SQL with temp views)

2、如何规划文件存储和布局：解决 负载不均衡 / 数据倾斜问题
我该怎么什么格式、压缩方案、分区方案呢？
文件格式：尽量用强类型的，推荐parquet
压缩方案：优先用可拆分的格式，推荐LZ4, BZip2, LZO，避免过大的、无法拆分的格式。
分区方案：看使用场景，要做权衡
    for read: 数据尽量按最常用的聚合维度来分区（比如日期）尽量避免shuffle.
    for write: 要考虑避免热点分区、数据倾斜、过多的小文件影响读性能
文件尺寸：repartition避免单个文件过大，避免过多小文件

File types:
- Prefer columnar over text for analytical queries
- Parquet
    – Column pruning
    – Predicate pushdown
- CSV/JSON
    – Must parse whole row
    – No column pruning
    – No predicate pushdown

Compression:
- Prefer splittable
    – LZ4, BZip2, LZO, etc.
- Parquet + Snappy or GZIP (splittable due to row groups) 
    – Snappy is default in Spark 2.2+
- AVOID 
    - Large GZIP text files
    – Not splittable
    – GC issues with wide tables
- More - [Why You Should Care about Data Layout in the Filesystem](https://databricks.com/session/why-you-should-care-about-data-layout-in-the-filesystem)


Partition:
PARTITIONING
• Coarse-grained filtering of input files
• Avoid over-partitioning (small files), overly nested partitions

BUCKETING
• Write data already hash-partitioned • Good for joins or aggregations
(avoids shuffle)
• Must be Datasource table

Writing Partitioned & Bucketed Data
• Each task writes a file per-partition and bucket
• If data is randomly distributed across tasks...lots of small files!
• Coalesce?
– Doesn’t guarantee colocation of partition/bucket values
– Reduces parallelization of final stage
• Repartition - incurs a shuffle but results in cleaner files
• Compaction - run job periodically to generate clean files

[分区与分桶的区别](https://stackoverflow.com/questions/56857453/what-is-the-difference-between-partitioning-and-bucketing-in-spark)：
- repartition is for using as part of an Action in the same Spark Job.
- bucketBy is for output, write. And thus for avoiding shuffling in the next Spark App, typically as part of ETL. Think of JOINs.

[Hive 分区与分桶](https://mikolaje.github.io/2019/hive_bucket_partition.html)
bucket在字段有很高的cardinality，和在数据在不同bucket中均匀地分布的时候，会表现出优越性。
partition在cardinality partition的字段不是很多的时候，会表现出优越性。


What If Partitions Are Too Big?
• Spark 2.2 - spark.sql.files.maxRecordsPerFile (default disabled) • Write task is responsible for splitting records across files
– WARNING: not parallelized
• If need parallelization - repartition with additional key
– Use hash+mod to manage # of files
 

3、sql优化：shuffle是第一大性能杀手，减少shuffle、本地计算
df.explain看看DAG计算图
spark现在有了AQE机制，能够按照DAG自动做优化。

可能触发shuffle的操作：sortBy, groupBy, repartition, join, window
说“可能”是因为有的partition可能已经是按groupBy分区的。

Spark SQL Shuffle Partitions
- Default to 200, used in shuffle operations – groupBy, repartition, join, window
- Depending on your data volume might be too small/large 
- Override with conf - spark.sql.shuffle.partitions
    – 1 value for whole query
 

4、 