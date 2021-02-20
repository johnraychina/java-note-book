
Facebook VS  Whatsapp signal

Facebook：一直保留，除非手动清理。
Whatsapp：端到端加密，未读信息存服务端，已读信息存推送到客户端后就移除。


# 对Facebook的设想
Facebook功能特性和设计主题：
- one to one chatting
- online /sent /read
- sending pictures or other files
- database
- security


User A <--> Load Balancer <--> Server Nodes <--> Database(Cassandra) + Cache(Redis)
User B <--> Load Balancer <--> Server Nodes <--> Database(Cassandra) + Cache(Redis)

## 登录
User A --> Node1
User B --> Node2

## 消息格式
message {
    sender: User A
    reciever: User B
    body: "Hi"  / 对象存储OSS url
    format: txt / jpg...
}

## 发消息
网络模型：A --(message)--> Node1
网络协议：Http, Websocket, Bosh, Long polling http
负载均衡 与会话保持（Load Balancing and Session Keeping）


## 两个服务器衔接
Redis:
User   | Server | heartbeat time
User A | Node 1 | xxxx
User B | Node 2 | xxxx

网络模型：Server Node1 --> Server Node2

## 收消息
网络模型： Node2 --(message)--> B


对话控制表： { users, conversationId, encryptionKey }
对话消息表： { conversationId, seq, message}
用户控制表： { user, conversationId, offset}

怎么消息是否已经收到？
由于消息的有序性，如果第i条消息已收到，那么前面的消息肯定是收到了。
所以对于一个对话，一个用户只需要记录最后一次收到的消息位置。

时间与序号


## 消息搜索


# 对Whatsapp的设想

先看需求

功能性需求

非功能性需求：
- 安全
- 数据容量、吞吐量、延迟
- CAP：一致性、可用性


## 数据模型

### user 客户端的 【发送队列】列表：
对话1发送队列：发送偏移量（客户端offset）, msg, msg, msg，
对话2发送队列：发送偏移量（客户端offset）, msg, msg, msg，

作用是：断网这段时间内发的消息，存在客户端上，记录偏移量


### user 客户端的 【接收队列】列表：
对话1：msg, msg, msg，接收偏移量（服务端消息序号）
对话2：msg, msg, msg，接收偏移量（服务端消息序号）

### 服务端
对话控制表（+缓存）：对话id，参与者，加密key，最后一条消息序号
对话消息日志：对话id，消息序号，消息格式，消息内容，发送人，接收人，接收状态


### 断网重连
打开对话，接收消息时：客户端请求服务端，服务端计算有哪些消息：where msg_seq:[客户端接收偏移量 ~ 最后一条消息序号] and reciver=userId

打开对话，由于之前断网，消息暂存在客户端本地的发送队列，现在自动发送：发送成功后，服务端返回一个消息序号，从发送队列-->转移到接收队列

客户端看到的对话消息，是一个合成队列 = 接收队列 + 发送队列

可以看到：
- 如果对话太多（比如一个用户有N个对话），做拉取时（读扩散），对存储的读取就放大N倍。
- 那么不做读扩散，而是写扩散呢？一个群如果有M个人，那么要写M个队列，对存储的写放大M倍。

微信是写扩散，所以最大群是500人。

钉钉最开始也是写扩散，后来为了支持万人群，做了读扩散。


# 微信
Logicsvr：
逻辑层的业务逻辑服务最早只有一个服务模块（我们称之为mmweb），囊括了所有提供给客户端访问的API，甚至还有一个完整的微信官网。
这个模块架构类似Apache，由一个CGI容器（CGIHost）和若干CGI组成（每个CGI即为一个API），不同之处在于每个CGI都是一个动态库so，由CGIHost动态加载。
在mmweb的CGI数量相对较少的时候，这个模块的架构完全能满足要求，但当功能迭代加快，CGI量不断增多之后，开始出现问题
于是我们开始尝试使用一种新的CGI架构——Logicsvr。

Svrkit框架：Qos过载保护FastReject设计

KVSvr：
单个Master-Slave分组中，Master还是单点，无法提供实时的写容灾，也就意味着无法消除单点故障。
另外Master-Slave的流水同步延时对读服务有很大影响，流水出现较大延时会导致业务故障。
于是我们寻求一个可以提供高性能、具备读写水平扩展、没有单点故障、
可同时具备读写容灾能力、能提供强一致性保证的底层存储解决方案，最终KVSvr应运而生。


Master-Master 存储架构：
国内数据中心 - 海外数据中心，数据同步：
最终我们选择了跟Yahoo!的PNUTS系统类似的解决方案，需要对用户集合进行切分，国内用户以国内上海数据中心为Master，所有数据写操作必须回到国内数据中心完成；海外用户以海外数据中心为Master，写操作只能在海外数据中心进行。从整体存储上看，这是一个Master-Master的架构，但细到一个具体用户的数据，则是Master-Slave模式，每条数据只能在用户归属的数据中心可写，再异步复制到其他数据中心。

