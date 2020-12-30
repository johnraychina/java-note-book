
连接器 > 分析器 > 优化器 > 执行器 > 存储引擎

InnoDB与MyISAM存储引擎对比
第一个重大区别是InnoDB的数据文件本身就是索引文件。
MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。
而在InnoDB中，表数据文件本身就是按B+Tree组织的一个索引结构，这棵树的叶节点data域保存了完整的数据记录。
这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。
 
第二个与MyISAM索引的不同是InnoDB的辅助索引data域存储相应记录主键的值而不是地址。
换句话说，InnoDB的所有辅助索引都引用主键作为data域。

## 持久化保证
https://time.geekbang.org/column/article/76161

WAL(write ahead log)先写日志机制：
redo log 和 binlog 保证持久化到磁盘，就能确保 MySQL 异常重启后，数据可以恢复。

### redo log(InnoDB)
如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程 IO 成本、查找成本都很高。
为了解决这个问题，MySQL 的设计者就用了类似酒店掌柜粉板的思路来提升更新效率。
而粉板和账本配合的整个过程，其实就是 MySQL 里经常说到的 WAL 技术，WAL 的全称是 Write-Ahead Logging.

具体来说，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log（粉板）里面，并更新内存，这个时候更新就算完成了。
同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做，这就像打烊以后掌柜做的事。
如果今天赊账的不多，掌柜可以等打烊后再整理。

但如果某天赊账的特别多，粉板写满了，又怎么办呢？
这个时候掌柜只好放下手中的活儿，把粉板中的一部分赊账记录更新到账本中，然后把这些记录从粉板上擦掉，为记新账腾出空间。与此类似，InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。

### bin log(MySQL Server)
为什么会有两份日志呢？因为最开始 MySQL 里并没有 InnoDB 引擎。
MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。
而 InnoDB 是另一个公司以插件形式引入 MySQL 的，既然只依靠 binlog 是没有 crash-safe 能力的，所以 InnoDB 使用另外一套日志系统——也就是 redo log 来实现 crash-safe 能力。

这两种日志有以下三点不同:
- redo log 是 InnoDB 引擎特有的；
- binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
- redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

### 2阶段提交
写数据时：更新内存 >  redolog(prepare)  > 写binlog >  redolog(commit) 
为什么必须有“两阶段提交”呢？这是为了让两份日志之间的逻辑一致。


我们先来看一下崩溃恢复时的判断规则：
如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；
如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：
    a. 如果是，则提交事务；
    b. 否则，回滚事务。
以此来保证主备一致性。

追问 2：redo log 和 binlog 是怎么关联起来的?
回答：它们有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log。

### 三种复制模式
异步复制（Asynchronous replication）：MySQL默认的复制即是异步的，主库在执行完客户端提交的事务后会立即将结果返给给客户端，并不关心从库是否已经接收并处理，这样就会有一个问题，主如果crash掉了，此时主上已经提交的事务可能并没有传到从上，如果此时，强行将从提升为主，可能导致新主上的数据不完整。

全同步复制（Fully synchronous replication）：指当主库执行完一个事务，所有的从库都执行了该事务才返回给客户端。因为需要等待所有从库执行完该事务才能返回，所以全同步复制的性能必然会收到严重的影响。

半同步复制（Semisynchronous replication）：介于异步复制和全同步复制之间，主库在执行完客户端提交的事务后不是立刻返回给客户端，而是等待至少一个从库接收到并写到relay log中才返回给客户端。相对于异步复制，半同步复制提高了数据的安全性，同时它也造成了一定程度的延迟，这个延迟最少是一个TCP/IP往返的时间。所以，半同步复制最好在低延时的网络中使用。

### 半同步复制潜在问题
假设客户端事务在存储引擎层提交后，在得到从库确认的过程中，主库宕机了。此时，可能存在以下两种情况：

事务还没发送到从库上：此时，客户端会收到事务提交失败的信息，客户端会重新提交该事务到新的主上，当宕机的主库重新启动后，以从库的身份重新加入到该主从结构中，会发现，该事务在从库中被提交了两次，一次是之前作为主的时候，一次是被新主同步过来的。

事务已经发送到从库上：此时，从库已经收到并应用了该事务，但是客户端仍然会收到事务提交失败的信息，重新提交该事务到新的主上。

## change buffer
 
 
## mysql多版本管理
https://time.geekbang.org/column/article/68963
https://blog.jcole.us/2014/04/16/the-basics-of-the-innodb-undo-logging-and-history-system/
https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html


mysql使用 roll back segment(undolog ) 记录被修改数据的老版本
一方面用于事务回滚
一方面用于事务隔离

mysql的每条记录有4个隐藏字段：
6-byte DB_TRX_ID 最后改动（包括删除操作）这行数据的事务id
a special bit: 删除标志
7-byte DB_ROLL_PTR 回滚指针，指向一条undolog
6-byte DB_ROW_ID 行号， clustered index聚簇索引会包含它

回滚段中的undolog分为insert和update undologs
insert undolog 插入事务提交后就被废弃
update undolog 更新事务提交后，还不能废弃，若有一致性读事务产生这条记录快照时，要给这些事务提供早期版本，所以要等这些事务全都不在时才能废弃。

所以不能把事务拖得太长（包括哪些只读事务），否则InnoDB无法将对应undologs的数据清空，导致rollback segment回滚段被打满。
Commit your transactions regularly, including those transactions that issue only consistent reads. Otherwise, InnoDB cannot discard data from the update undo logs, and the rollback segment may grow too big, filling up your tablespace.


## 索引与锁
在 InnoDB 中，表都是根据主键顺序以索引的形式存放的，这种存储方式的表称为索引组织表。又因为前面我们提到的，InnoDB 使用了 B+ 树索引模型，所以数据都是存储在 B+ 树中的。

常见索引结构：
- 哈希表这种结构适用于只有等值查询的场景，比如 Memcached 及其他一些 NoSQL 引擎。
但缺点是，因为不是有序的，所以哈希索引做区间查询的速度是很慢的。

- 而有序数组在等值查询和范围查询场景中的性能就都非常优秀。
如果仅仅看查询效率，有序数组就是最好的数据结构了。但是，在需要更新数据的时候就麻烦了，你往中间插入一个记录就必须得挪动后面所有的记录，成本太高。所以，有序数组索引只适用于静态存储引擎，比如你要保存的是 2017 年某个城市的所有人口信息，这类不会再修改的数据。

- 树
二叉树是搜索效率最高的，但是实际上大多数的数据库存储却并不使用二叉树。其原因是，索引不止存在内存中，还要写到磁盘上。为了让一个查询尽量少地读磁盘，就必须让查询过程访问尽量少的数据块。那么，我们就不应该使用二叉树，而是要使用“N 叉”树。这里，“N 叉”树中的“N”取决于数据块的大小。

以 InnoDB 的一个整数字段索引为例，这个 N 差不多是 1200。这棵树高是 4 的时候，就可以存 1200 的 3 次方个值，这已经 17 亿了。考虑到树根的数据块总是在内存中的，一个 10 亿行的表上一个整数字段的索引，查找一个值最多只需要访问 3 次磁盘。其实，树的第二层也有很大概率在内存中，那么访问磁盘的平均次数就更少了。
N 叉树由于在读写上的性能优点，以及适配磁盘的访问模式，已经被广泛应用在数据库引擎中了。


### 聚簇索引与非聚簇索引
主键索引的叶子节点存的是整行数据。在 InnoDB 里，主键索引也被称为聚簇索引（clustered index）。
非主键索引的叶子节点内容是主键的值。在 InnoDB 里，非主键索引也被称为二级索引（secondary index)。
显然，主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。

InnoDB基于主键索引和普通索引的查询有什么区别？
普通索引叶子节点不是记录，而是回表。
回到主键索引树搜索的过程，我们称为回表。


### 最左前缀
why：树形结构，compare对比时按字段从左到右匹配
### 覆盖索引
覆盖索引为啥快？不用回表，只查索引本身就行。

### 索引下推
满足最左前缀原则的时候，最左前缀可以用于在索引中定位记录。
这时，你可能要问，那些不符合最左前缀的部分，会怎么样呢？

MySQL 5.6 引入的索引下推优化（index condition pushdown)， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。

### 锁
全局锁、表级锁、行锁
#### 全局锁
全库备份时FTWRL(Flush Tables With Read Lock) 全库只读，阻塞所有变更。
官方自带的逻辑备份工具是 mysqldump。当 mysqldump 使用参数–single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。
而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

#### 表级锁
表锁: lock tables...read/write

元数据锁MDL(Meta Data Lock):
MDL 不需要显式使用，在访问一个表的时候会被自动加上。
MDL 的作用是，保证读写的正确性。如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

如何安全alter table而不影响业务？
比较理想的机制是，在 alter table 语句里面设定等待时间:
如果在这个指定的等待时间里面能够拿到 MDL 写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。
之后开发人员或者 DBA 再通过重试命令重复这个过程。



#### 行锁
行锁是引擎层面实现的，MyISAM就不支持行锁，这意味着并发效率低下

record lock 这个record是指索引记录

row lock 才是行锁

## 主备切换
CAP 原理，想要保持C，就需要停止服务（降低A），确保同步完成再online.


### 基于点位的主备切换
change master命令执行切换，一共6个参数：
CHANGE MASTER TO 
MASTER_HOST=$host_name 
MASTER_PORT=$port 
MASTER_USER=$user_name 
MASTER_PASSWORD=$password 
MASTER_LOG_FILE=$master_log_name 
MASTER_LOG_POS=$master_log_pos  



### 基于GTID切换
一主多从的主备切换流程。在这个过程中，从库找新主库的位点是一个痛点。
由此，我们引出了 MySQL 5.6 版本引入的 GTID 模式，介绍了 GTID 的基本概念和用法。
在 GTID 模式下，一主多从切换就非常方便了。
我们还能看到 GTID 模式在读写分离场景的应用。


## 数据安全

# mysql高可用

## 读写分离，如何实现？
多数据源 + 切面
中间件TDDL

## Mysql Router 
https://dev.mysql.com/doc/mysql-router/8.0/en/mysql-router-installation-osx.html


## 传统主从复制问题
基于异步复制的高可用方案存在主从不一致乃至丢失事务的风险，原因在于当master将事务写入binlog，然后复制给slave后并不等待slave回复即进行提交，若slave因网络延迟或其它问题尚未收到binlog日志，而此时master故障，应用切换到slave时，本来在master上已经提交的事务就会丢失，因其尚未传送到slave，从而导致主从之间事务不一致。

基于semi-sync复制的高可用方案也存在主备不一致的风险，原因在于当master将事务写入binlog，尚未传送给slave时master故障，此时应用切换到slave，虽然此时slave的事务与master故障前是一致的，但当主机恢复后，因最后的事务已经写入到binlog，所以在master上会恢复成已提交状态，从而导致主从之间的事务不一致。

## Group Replication 分组复制应运而生（5.7引入）
http://mysql.taobao.org/monthly/2017/08/01/
https://dev.mysql.com/doc/refman/8.0/en/group-replication.html

- 不能基于语句：不确定性，不安全。Any nondeterministic behavior is difficult to replicate when using statement-based replication.

- 必须基于行：确定性，锁的粒度小。

- 基于Paxos共识算法的xcom协议，保证半数以上节点可用即可。

需要注意的是，group组内的通讯是采用基于paxos协议的xcom来实现的，
它的一个特性就是消息是有序传送，每个节点接收到的消息顺序都是相同的，并且至少保证半数以上节点收到才会认为消息发送成功。
xcom的这些特性对于数据库状态机来说非常重要，是保证数据库状态机一致性的关键因素。 

## 核心组件XCOM的特性

- 闭环(closed group)：只有组内成员才能给组成员发消息，不接受外部成员消息。
- 消息全局有序(total order)：所有XCOM传递的消息都是全局有序（在多主集群中或是偏序），这是构建一致性状态机的基础。
- 消息安全送达(safe delivery)：必须多数节点确认才通知上层应用。
- 视图同步(view synchrony)：在成员视图变化之前，每个节点都以相同的顺序传递消息，这保证在节点恢复时有一个同步点。实际上，组复制并不强制要求消息传递必须在同一个节点视图中。





