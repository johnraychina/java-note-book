


# 什么是zookeeper?
Zookeepr是一个分布式的协调服务，服务于分布式系统。
本质上是一个分布式存储+协调组件。


# zookeeper能干什么？
命名服务：类似于电话本，比如服务注册
leader选举
锁：实现分布式互斥锁
同步：pub/sub发布订阅
配置管理

# 如何使用zookeeper

## 安装配置

## 集群配置
https://cloud.tencent.com/developer/article/1021121

1. 多份conf配置，dataDir数据文件位置 和 clientPort端口改一下即可

conf1/zoo.cfg
```text
dataDir=./data1
clientPort=2181
server.1=localhost:8880:7770
server.2=localhost:8881:7770
server.3=localhost:8882:7770
```

2. cd zookeeper主目录, 启动多个zookeeper实例（进程）
```sh
./bin/zkServer.sh --config ./conf1 start
./bin/zkServer.sh --config ./conf2 start
./bin/zkServer.sh --config ./conf3 start
```

3. 查看状态
```sh
$ ./bin/zkServer.sh --config ./conf1 status
Using config: ./confN/zoo.cfg
Mode: follower 或者是 leader
```

## zookeeper四字命令
在zoo.cfg配置中启用四字命令需要加白名单：
4lw.commands.whitelist=*

- conf 查看配置
- cons 查看链接
- dump 只能在leader节点上用
- envi 查看环境
- ruok Are you ok?
- stat 查看统计信息
- srst Reset统计信息
- dirs 查看快照和日志大小
- wchs 查看服务器watch信息
- wchp 查看服务器watch信息，它的输出一个与session相关的路径。
$ echo stat | nc loclahost 2181

## 客户端命令




## java调用


# zookeeper原理
https://www.cnblogs.com/leesf456/p/6107600.html
https://zookeeper.apache.org/doc/r3.6.2/zookeeperInternals.html

- 原子广播 Atomic Broadcast
- 一致性保障 Consistency Guarantees
- 仲裁 Qurums
- 日志 Logging





## 消息广播
2PC提交：
zookeeper使用单一的leader来接收和处理客户端的所有事物请求，
并采用ZAB协议的原子广播协议，将事物请求以proposal广播给所有followers，
leader收到过半ACK反馈时，leader再次将commit消息发给followers。

注意：Observer只负责同步数据，不参与2PC数据同步过程。

## 崩溃恢复
何时执行崩溃恢复？
leader宕机时
leader与超过半数节点通信失败时

崩溃恢复需要通过ZAB协议重新选举leader节点。

### ZAB协议如何避免选出多个leader？
ZAB本质是是一个multi-paxos算法
ZAB一个周期叫做epoch
ZAB通过paxos算法选举出leader后就按leader的命令写入数据。
epoch > serverId


raft一个周期叫叫term
raft保持日志连续性，心跳方向是leader->follower，ZAB相反。


### 使用场景
https://zookeeper.apache.org/doc/r3.1.2/recipes.html#sc_recipes_twoPhasedCommit
- 开箱即用：命名服务，配置存储，分组
- 栅栏
- 队列
- 锁
- leader选举
- 二阶段提交

#### 栅栏
Distributed systems use barriers to block processing of a set of nodes until a condition is met at which time all the nodes are allowed to proceed. Barriers are implemented in ZooKeeper by designating a barrier node. The barrier is in place if the barrier node exists. Here's the pseudo code:

1. Client calls the ZooKeeper API's exists() function on the barrier node, with watch set to true.

2. If exists() returns false, the barrier is gone and the client proceeds

3. Else, if exists() returns true, the clients wait for a watch event from ZooKeeper for the barrier node.

When the watch event is triggered, the client reissues the exists( ) call, again waiting until the barrier node is removed.

#### 队列：pub/sub发布订阅，实现类似队列的功能
Distributed queues are a common data structure. To implement a distributed queue in ZooKeeper, first designate a znode to hold the queue, the queue node. The distributed clients put something into the queue by calling create() with a pathname ending in "queue-", with the sequence and ephemeral flags in the create() call set to true. Because the sequence flag is set, the new pathnames will have the form _path-to-queue-node_/queue-X, where X is a monotonic increasing number. A client that wants to be remove from the queue calls ZooKeeper's getChildren( ) function, with watch set to true on the queue node, and begins processing nodes with the lowest number. The client does not need to issue another getChildren( ) until it exhausts the list obtained from the first getChildren( ) call. If there are are no children in the queue node, the reader waits for a watch notification to check to queue again.

Priority Queues
To implement a priority queue, you need only make two simple changes to the generic queue recipe . First, to add to a queue, the pathname ends with "queue-YY" where YY is the priority of the element with lower numbers representing higher priority (just like UNIX). Second, when removing from the queue a client uses an up-to-date children list meaning that the client will invalidate previously obtained children lists if a watch notification triggers for the queue node.

#### 锁：采用临时节点实现分布式互斥锁


```Java
//入口 org.apache.curator.framework.recipes.locks.InterProcessMutex#acquire()
CuratorFramework zkClient = getZkClient();
String lockPath = "/mynode";
InterProcessMutex lock = new InterProcessMutex(zkClient, lockPath);
interProcessMutex.acquire()
//一直看下去，就是对比临时节点index位置，和需要获取锁个数maxLeases
//由于加锁的顺序性，如果当前是第一个节点，说明获得锁成功，后续加锁请求只会在后面加节点
boolean getsTheLock = ourIndex < maxLeases;
```

Maven 依赖
```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.1.0</version>
</dependency>
```

**加锁逻辑**
1. Call create( ) with a pathname of "_locknode_/lock-" and the sequence and ephemeral flags set.
2. Call getChildren( ) on the lock node without setting the watch flag (this is important to avoid the herd effect).
3. If the pathname created in step 1 has the lowest sequence number suffix, the client has the lock and the client exits the protocol.
4. The client calls exists( ) with the watch flag set on the path in the lock directory with the next lowest sequence number.
5. if exists( ) returns false, go to step 2. Otherwise, wait for a notification for the pathname from the previous step before going to step 2.

**解锁**
The unlock protocol is very simple: clients wishing to release a lock simply delete the node they created in step 1.

**注意**
1. The removal of a node will only cause one client to wake up since each node is watched by exactly one client. In this way, you avoid the herd effect.
2. There is no polling or timeouts.
3. Because of the way you implement locking, it is easy to see the amount of lock contention, break locks, debug locking problems, etc.


#### leader选举
非常简单：创建 SEQUENCE|EPHEMERAL节点，如果是最小节点，则成为leader.

需要注意的是：为了避免重新选举leader导致herd effect，每个节点只顺序监听上个节点，而不是所有节点监听一个leader。

Here's the pseudo code:

1. Let ELECTION be a path of choice of the application. To volunteer to be a leader:
Create znode z with path "ELECTION/n_" with both SEQUENCE and EPHEMERAL flags;

2. Let C be the children of "ELECTION", and i be the sequence number of z;

3. Watch for changes on "ELECTION/n_j", where j is the smallest sequence number such that j < i and n_j is a znode in C;

Upon receiving a notification of znode deletion:

1. Let C be the new set of children of ELECTION;

2. If z is the smallest node in C, then execute leader procedure;

3. Otherwise, watch for changes on "ELECTION/n_j", where j is the smallest sequence number such that j < i and n_j is a znode in C;

Note that the znode having no preceding znode on the list of children does not imply that the creator of this znode is aware that it is the current leader. Applications may consider creating a separate to znode to acknowledge that the leader has executed the leader procedure.

#### 二阶段提交
Two-phased Commit
A two-phase commit protocol is an algorithm that lets all clients in a distributed system agree either to commit a transaction or abort.

In ZooKeeper, you can implement a two-phased commit by having a coordinator create a transaction node, say "/app/Tx", and one child node per participating site, say "/app/Tx/s_i". When coordinator creates the child node, it leaves the content undefined. Once each site involved in the transaction receives the transaction from the coordinator, the site reads each child node and sets a watch. Each site then processes the query and votes "commit" or "abort" by writing to its respective node. Once the write completes, the other sites are notified, and as soon as all sites have all votes, they can decide either "abort" or "commit". Note that a node can decide "abort" earlier if some site votes for "abort".

An interesting aspect of this implementation is that the only role of the coordinator is to decide upon the group of sites, to create the ZooKeeper nodes, and to propagate the transaction to the corresponding sites. In fact, even propagating the transaction can be done through ZooKeeper by writing it in the transaction node.

There are two important drawbacks of the approach described above. One is the message complexity, which is O(n²). The second is the impossibility of detecting failures of sites through ephemeral nodes. To detect the failure of a site using ephemeral nodes, it is necessary that the site create the node.

To solve the first problem, you can have only the coordinator notified of changes to the transaction nodes, and then notify the sites once coordinator reaches a decision. Note that this approach is scalable, but it's is slower too, as it requires all communication to go through the coordinator.

To address the second problem, you can have the coordinator propagate the transaction to the sites, and have each site creating its own ephemeral node.


### Raft






