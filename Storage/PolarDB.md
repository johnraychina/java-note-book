 
http://mysql.taobao.org/monthly/2020/12/01/
http://mysql.taobao.org/monthly/2020/12/04/
https://help.aliyun.com/document_detail/148660.html

关键技术：RDMA, Partition ALL/ By Engine/ By Storage，  trade-off 通用性-可扩展性


# Partition ALL 分库分表
    

# Partition By Engine 按引擎分区
向用户屏蔽了分布式事务等细节，提供统一的数据库服务，简化了用户使用。

代表性数据库：Spanner, CockroachDB, Oceanbase, TIDB

# Partition By Storage 按存储分区
以Aurora及PolarDB为代表

## PolarDB

2017年，由于RDMA的出现及普及，大大加快了网络间的网络传输速率，PolarDB[15]认为未来网络的速度会接近总线速度，也就是瓶颈不再是网络，而是软件栈。因此PolarDB采用新硬件结合Bypass Kernel的方式来实现高效的共享盘实现，进而支撑高效的数据库服务。由于PolarDB的分片层次更低，也就能做到更好的生态兼容，也就是为什么PolarDB能够很快的做到社区版本的全覆盖。副本间PoalrDB采用了ParalleRaft来允许一定范围内的乱序确认，乱序Commit以及乱序Apply。


采用Partition Storage的策略的NewSQL，由于保持了完整的计算层，所以相对于传统数据库需要用户感知的变化非常少，能过做到更大程度的生态兼容。同时也因为只有存储层做了分片和打散，可扩展性不如上面提到的两种方案。在计算存储分离的基础上，Microsoft的Socrates[13]提出了进一步将Log模块拆分，实现Durability和Available的分离；Oracal的Cache Fusion[14]通过增加计算节点间共享的Memory来获得多点写及快速Recovery，总体来讲他们都属于Partition Storage这个范畴。