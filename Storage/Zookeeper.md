


# 什么是zookeeper?
Zookeepr是一个分布式的协调服务，服务于分布式系统。
本质上是一个分布式存储+协调组件。


# zookeeper能干什么？
leader选举
命名服务：类似于电话本，比如服务注册
锁：实现分布式互斥锁
同步：生产者-消费者如何协调的问题
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


### Raft






