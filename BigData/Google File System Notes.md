
# 
Client-Master-ChunkServer
- Master: 与 ChunkServer 通信，记录Metadata（包含File namespace文件目录和Chunk字典），接收Client的请求让其重定向到 ChunkServer.
- ChunkServer: 保存Chunk文件数据
- Chunk: 一个典型的chunk size是64MB的大小，chunk id用一个64bit(8B)的数字可以表示 8M * 64MB = 512 PB数据空间


前面提到GFS通过协同应用程序和GFS API来放松了对一致性的要求，
那GFS到底提供了什么样的一致性？

GFS保证：对文件命名空间（文件元数据）的操作是原子性的，比如文件创建。
Master节点通过对命名空间加锁，保证文它的原子性和正确性；
Master的操作日志 维护了对文件命名空间（文件元数据）操作的全局全量操作记录。

# 2.7 Consistency Model
File namespace mutations (e.g., file creation) are atomic. They are handled exclusively by the master: namespace locking guarantees atomicity and correctness (Section 4.1); the master’s operation log defines a global total order of these operations (Section 2.6.3).

consistent 一致的: 所有客户端看到的数据都是相同的。
defined 确定的: 所有的客户端并发的写入都没有互相干扰（也就是说没有互相覆盖），所有客户端都能看到自己写入的修改。

对于按偏移量写入(Write)：
- 多客户端顺序成功：defined
- 客户端并发写入成功：consitent but undefined
- 写入失败：inconistent
对于GFS的按偏移量写入来说，defined是更加严格的一致性，并发写入通常满足consistent，但通常是undefined：


对于记录追加(Record Append):
- 多客户端顺序成功:  defined interspersed with inconsistent
- 客户端并发写入成功: defined interspersed with inconsistent
- 写入失败：inconistent

## 2.7.2 Implications for Applications
GFS applications can accommodate the relaxed consis- tency model with a few simple techniques already needed for other purposes: relying on appends rather than overwrites, checkpointing, and writing self-validating, self-identifying records.

Practically all our applications mutate files by appending rather than overwriting. 

#### Write and Checkpoint 
In one typical use, a writer generates a file from beginning to end. It atomically renames the file to a permanent name after writing all the data, or periodically checkpoints how much has been successfully written. Checkpoints may also include application-level checksums. Readers verify and process only the file region up to the last checkpoint, which is known to be in the defined state. 

#### Read and Checkpoint
A reader can identify and discard extra padding and record fragments using the checksums. 
If it cannot tolerate the occasional duplicates, it can filter them out using unique identifiers in the records, which are often needed anyway to name corre-sponding application entities such as web documents. 

# 3.1 租约顺序和修改顺序 Leases and Mutation Order
对于多副本并发写操作，如何避免互相干扰？ ==> 全局序号 ==> 如何生成全局序号？

租约机制最小化master的开销。
副本节点中有个主节点（master挑选出来的），主节点从master得到租约序号（一般是60s），再自己维护一个修改序号，合起来作为全局修改序号。

# 3.2 数据流 Data Flow
网络传输性能最大化：
- 数据流控制流解耦
- chunkserver 链式推送数据，避免单一机器成为瓶颈
- 推送数据采用最近原则（根据网络ip算出来）
- 在 TCP  管道数据传输，最小化延迟

# 3.3 原子性地追加记录 Atomic Record Appends
并发写入同一个文件，如何避免互相覆盖？

**Record append**
客户端只管提供要写入的数据，GFS会预计算需要的空间，写入的offset位置由GFS计算给出。
GFS保证至少原子性追加成功一次，这类似Unix并发写入文件而不产生竞态的O_APPEND模式。



# 3.4 快照 Snapshot 


# 4.5 Stale Replica Detection

# 5.1 High Availability

Fast Recovery
Chunk Replication
Master Replication

**checksum 校验和**
**parity 奇偶校验**
**erasure codes 抹除码**

# 5.2 Data Integrity

