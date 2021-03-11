
由浅入深，先动手学习：
- 基本介绍
- 安装使用
- 概念与架构
- 理论基础
- 高级功能与优化

# 基本介绍
https://developer.ibm.com/zh/technologies/analytics/articles/ba-cn-bigdata-hbase/
https://developer.ibm.com/zh/articles/ba-1604-hbase-develop-practice/

分布式可扩展大数据存储，半结构化，无类型，按byte存储，文件系统基于HDFS。

HBase本身只提供Java API，结合Apache Phoenix 则可以实现sql查询。
结合Hive 可以实现HQL查询，但基于MapReduce，实时性不高。



对比传统数据库的ACID保证，
HBase提供单行Atomic，没有schema所以无Consistency保证，没有事务所以无Isolation保证，有Durable保证。
[ACID in HBase参见文档](http://hadoop-hbase.blogspot.com/2012/03/acid-in-hbase.html)


- Client: 客户端会先连接到zookeeper，读取并缓存META元数据，里面有Region信息，按META直连RegionServer做读写。
- Master: 对标BigTable的MasterServer，协调管理RegionServer，分配Region给RegionServer.
- Zookeeper: Master HA方案。并且负责Region和RegionServer的注册。
- RegionServer: 对标BigTable的TabletServer
- Region: 对标BigTable的Tablet
- MemStore: 对标BigTable的MemTable
- HFile: 对标BigTable的SSTable, HFile是Hadoop的二进制格式文件， StoreFile是对HFile轻量级包装，里面有一些元数据和索引。
- HLog File: 对标BigTable的CommitLog, HBase中WAL的存储格式，物理上是Hadoop的Sequence File.
- compaction: minor compaction, major compaction


# 安装使用
https://www.tutorialspoint.com/hbase/index.htm
https://hbase.apache.org/book.html#quickstart

## 单机部署 standalone

1. 下载解压hbase安装包，开启命令行，进入目录.
2. 配置: conf/hbase-env.sh
3. 启动: bin/start-hbase.sh
4. 命令行输入jps 查看HMaster进程是否启动
5. 进入管理页面 http://localhost:16010 

standalone单机模式下，HBase的所有后台守护线程都运行在一个JVM进程中（HMaster, HRegionServer, Zookeeper).


### 连接hbase
1. 命令行输入 
```
./bin/hbase shell
```
连接成功后会显示hbase的shell命令提示
```
hbase(main):001:0>
```

2. help显示hbase shell支持的命令
命令比较多，所以做了group分组。
namespace, ddl, dml 中的命令是比较常用的。

3. 创建表 create 

4. 显示表信息 list, describe

5. 操作数据
put 添加一条数据
get 查询一条数据
scan 扫描表

6. 禁用/启用表 disable/enable 

7. 删除表 drop

8. quit 退出hbase shell

### 停止hbase

```
$ ./bin/stop-hbase.sh
```
## 伪分布式部署（为了本地测试） 
Pseudo-Distributed for Local Testing

standalone单机模式下，HBase的所有后台模块（HMaster, HRegionServer, Zookeeper)都运行在一个JVM进程中.

你可以将HBase 重新配置成伪分布式模式：
HBase仍然运行在在一台主机内，而各个后台模块（HMaster, HRegionServer, Zookeeper)运行在独立的进程中。


**关于存储**
- 如果你已经有可用的HDFS，我们假设你的数据存在HDFS上；
[如何配置一个单节点Hadoop](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)

- 你也可以跳过HDFS的配置，直接将你的数据存在本地文件系统中；
默认情况下，你的HBase数据会存储于/tmp目录下，除非你配置了 hbase.rootdir.



**开始安装**
1. 停止HBase
因为后续步骤会创建新的目录来存储数据，所以你之前创建的数据库会丢失。

2. 配置HBase

2.1 hbase-site.xml 配置模式和hdfs URI
```xml
<property>
  <name>hbase.cluster.distributed</name>
  <value>true</value>
</property>

<property>
  <name>hbase.rootdir</name>
  <value>hdfs://localhost:8020/hbase</value>
</property>
```

2.2 去掉这2个配置项 
hbase.tmp.dir 
hbase.unsafe.stream.capability.enforce


3. 启动HBase
```
$ bin/start-hbase.sh
```

4. 检查HDFS中的HBase目录
如果一切顺利，HBase会在HDFS中创建自己的目录，根据上面的配置，目录位置是在/hbase

你可以用 hadoop fs 命令来查看目录
```
$ ./bin/hadoop fs -ls /hbase

drwxr-xr-x   - hbase users          0 2014-06-25 18:58 /hbase/.tmp
drwxr-xr-x   - hbase users          0 2014-06-25 21:49 /hbase/WALs
drwxr-xr-x   - hbase users          0 2014-06-25 18:48 /hbase/corrupt
drwxr-xr-x   - hbase users          0 2014-06-25 18:58 /hbase/data
-rw-r--r--   3 hbase users         42 2014-06-25 18:41 /hbase/hbase.id
-rw-r--r--   3 hbase users          7 2014-06-25 18:41 /hbase/hbase.version
drwxr-xr-x   - hbase users          0 2014-06-25 21:49 /hbase/oldWALs
```

5. 使用ddl创建表或者dml操作数据

6. 启动/关闭 备用HMaster
> 这里只是为了测试和学习，在一台主机上运行多个HMaster实例，对于生产环境来说没有实际意义。

**启动备用master**
HMaster会占用2个端口，默认是16000 and 16010,
我们这里启动3个备用HMaster实例，分别占用6个端口：16002/16012, 16003/16013, and 16005/16015。
其中，2 3 5 代表在16000的基础上的offset偏移量。
```
$ ./bin/local-master-backup.sh start 2 3 5
```
**关闭备用master**
想要只关闭备用HMaster而不是整个集群，只需要按进程关闭。
进程号就在类似于这样一个文件中： /tmp/hbase-USER-X-master.pid
```
$ cat /tmp/hbase-testuser-1-master.pid |xargs kill -9
```


7. 启动/关闭 额外的RegionServer
每个RegionServer实例占用2个端口，默是16020 and 16030.
和备用HMaster一样，也是指定端口的offset.
**启动**
```
$ .bin/local-regionservers.sh start 2 3 4 5
```

**关闭**
```
$ .bin/local-regionservers.sh stop 3
```


一般生产环境RegionServer比较多，如果10个不够用，你可以在启动前修改这2个设置：
```
HBASE_RS_BASE_PORT
HBASE_RS_INFO_BASE_PORT
```
譬如改成16200 and 16300, 那么可以支持100个RegionServer.

8. 关闭 HBase
```
$ bin/stop-hbase.sh
```


## 全分布式存储（用于生产环境）
最小集群3节点示例：
Master一主一备
ZooKeeper高可用最小集群3台主机
RegionServer多个实例

部署架构如下表：

| Node Name        | Master   | ZooKeeper   | RegionServer |
|------------------|----------|-------------|--------------|
|node-a.exmple.com | yes      | yes         | no           |
|node-b.exmple.com | backup   | yes         | yes          |
|node-c.exmple.com | no       | yes         | yes          |


以下安装部署步骤，假设每个节点都是一个虚拟机，并且都位于同一个网络下，
并且基于前面的 [伪分布式](##伪分布式部署（为了本地测试）) 作为起点，假设你前面部署的节点是 node-a.


> 请确保所有节点之间网络通畅，并且防火墙规则不会阻拦彼此访问。
> 如果你看到 no route to host 之类的报错，请检查你的防火墙设置。


### 配置无密码的SSH访问方式
为了启动后台服务，作为master，node-a需要能登录node-b, nod-c（当然也包括自己）。
最简单的方式就是使用相同的username，并且配置无密码的SSH登录方式：

1. 在node-a上，生成 key pair.  
使用运行HBase的用户登录节点，并用命令生成ssh key pair.  
运行成功后，命令行会显示key pair的输出目录，其中公钥默认名称是 id_rsa.pub . 
```
$ ssh-keygen -t rsa
```


2. 创建一个目录存储 shared keys  
在node-b和node-c上，使用运行HBase的用户登录，并且在用户home目录下创建一个.ssh 目录（如果不存在）。  
如果.ssh已经存在，注意它可能已经包含其他的key了。

3. 将公钥拷贝至其他节点  
将公钥从node-a拷贝至其他节点，为了保证安全，可以使用scp或其他安全方式传输。  
然后在其他节点（node-b,node-c)创建文件.ssh/authorized_keys, 并将公钥(id_rsa.pub)的内容追加到文件末尾：
```
$ cat id_rsa.pub >> ~/.ssh/authorized_keys
```

4. 测试ssh无密码登录  
从node-a ssh登录到其他节点（node-b, node-c）.

5. 由于node-b会运行备用master实例，所以你需要重复以上步骤，确保能从node-b ssh登录到其他节点。


### 准备节点node-a
按照前面的部署架构表，node-a要运行主master进程和ZooKeeper经常，但没有RegionServer.
所以先要停止node-a上的RegionServer.

1. 修改 conf/regionservers，删除包含localhost的行.
添加node-b, node-c的主机名称或ip地址。

2. 配置HBase，使用node-b作为备用master:  
在conf目录下创建 backup-masters 文件，添加备用master的主机名。  
在这个例子里面，对应是node-b的主机名：b.exmaple.com

conf/backup-masters:
```
b.exmaple.com
```

3. 配置ZooKeeper
你应该小心配置ZooKeeper，尤其是生产环境，详细参见[zookeeper部分](https://hbase.apache.org/book.html#zookeeper).

这个配置会引导HBase在集群的各个节点上启动并管理一个ZooKeeper实例。

conf/hbase-site.xml

```xml
<property>
  <name>hbase.zookeeper.quorum</name>
  <value>node-a.example.com,node-b.example.com,node-c.example.com</value>
</property>
<property>
  <name>hbase.zookeeper.property.dataDir</name>
  <value>/usr/local/zookeeper</value>
</property>
```


4. 将你配置中所有主机名为localhost的地方，全都改为其他节点引用本节点的主机名，在这个例子里面是 node-a.example.com


### 准备node-b, node-c

1. 下载解压HBase
2. 将node-a的conf目录拷贝到node-b, node-c，集群的各个节点保持相同的配置信息。


### 启动并测试你的集群

1. 确保HBase没有在任何节点上运行。
> jps看一下 有没有Master, HRegionServer, HQuorumPeer 进程在运行，有的话就kill掉.

2. 启动集群  
在主Master节点(node-a)上执行 start-hbase.sh, 你应该会看到类似下面的输出: 
```
$ bin/start-hbase.sh
node-c.example.com: starting zookeeper, logging to /home/hbuser/hbase-0.98.3-hadoop2/bin/../logs/hbase-hbuser-zookeeper-node-c.example.com.out
node-a.example.com: starting zookeeper, logging to /home/hbuser/hbase-0.98.3-hadoop2/bin/../logs/hbase-hbuser-zookeeper-node-a.example.com.out
node-b.example.com: starting zookeeper, logging to /home/hbuser/hbase-0.98.3-hadoop2/bin/../logs/hbase-hbuser-zookeeper-node-b.example.com.out
starting master, logging to /home/hbuser/hbase-0.98.3-hadoop2/bin/../logs/hbase-hbuser-master-node-a.example.com.out
node-c.example.com: starting regionserver, logging to /home/hbuser/hbase-0.98.3-hadoop2/bin/../logs/hbase-hbuser-regionserver-node-c.example.com.out
node-b.example.com: starting regionserver, logging to /home/hbuser/hbase-0.98.3-hadoop2/bin/../logs/hbase-hbuser-regionserver-node-b.example.com.out
node-b.example.com: starting master, logging to /home/hbuser/hbase-0.98.3-hadoop2/bin/../logs/hbase-hbuser-master-nodeb.example.com.out
```

首先ZooKeeper集群启动，接着是master，然后是RegionServer，最后启动备用master.

3. 验证各节点进程是否运行

node-a:
```
$ jps
20355 Jps
20071 HQuorumPeer
20137 HMaster
```

node-b
```
$ jps
15930 HRegionServer
16194 Jps
15838 HQuorumPeer
16010 HMaster
```

node-c
```
$ jps
13901 Jps
13639 HQuorumPeer
13737 HRegionServer
```

> 这里ZooKeeper进程名是HQuorumPeer，它是受HBase控制。

4. 打开web控制页面
主master：http://node-a.example.com:16010/
备用master：http://node-b.example.com:16010/

5. kill某个进程，看看某个节点或服务挂掉会发生什么。   
三节点的集群还不是非常健壮，只能容忍1个节点挂掉。



## HBase详细配置
参见 https://hbase.apache.org/book.html#configuration



## TODO: K8s部署

