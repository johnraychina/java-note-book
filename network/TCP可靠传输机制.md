
https://coolshell.cn/articles/11609.html

## TCP 可靠传输

- 拥塞控制 Congestion Control
- 流量控制 Flow Control
- 连接的建立和关闭机制 Connection Setup

### 从哪里来到哪里去？
TCP逻辑上是进程Process（port）之间的通信。
TCP层将字节流切分为segment，加上TCP的header

IP层逻辑上是主机之间的通信Host(ip)
IP层加上IP的header成为packet

### 如何解决网络链路中不可靠的问题
#### Problem1: packet corruption
Detect: checksum/CRC
Solution: retransmit

#### Problem2: packet loss
Detect: ACK + sequence number + Timeout
Solution: retransmit

#### Problem3: packet duplicate
Detect: ACK + sequence number
Solution: drop

#### Problem4: stop-and-wait causes low bandwith uitilty
Solution: pipeline protocols: buffering + Range of sequence number of packets each time

### Problem5: lost ACK
Solution: resend packet

### Problem6: premature timeout
Solution: accumulative ACK

#### Retransmit 综合前面4个问题来看重传方案：
- Go-Back-N 只有一个Timer，中间一个包丢了，从中间开始重传后续的包
- Selective Repeat 多个Timer，中间一个包丢了，只重传丢的那个包，由selective ack功能支持
timeout超时重传太慢了，有没有有更好的办法呢？
Fast Retransmit快速重传机制：通过3个ACK返回同一个sequence number则认为包丢了，触发重传。


### TCP滑动窗口
可靠传输：保证数据送达，如果未到达，能够发现并重传。
控制拥塞：管理数据的发送速率，不至于接收数据过载。

Reciver计算 rwnd = RcvBuffer - (LastByteRcvd - LastByteRead) 
通过ACK通知Sender
Sender限制可用发送窗口：LastByteSent - LastByteAcked <= rwnd

设备A 发送消息给 设备B
每条消息设置一个识别号，可以被独立确认。同一时刻可以发送多条消息。
设备B定期给A发送一条限制参数，制约设备A一次可发送的消息最大量。
片段发送确认，最后一个字节sequence number：提高速度。

**TCP数据流状态**
1. 已发送 已确认
2. 已发送 未确认
3. 未发送 接收方ready
4. 未发送 接收方not ready

**sequence number设定与同步**

发送窗口：已发送未确认+未发送接收方ready
可用窗口：考虑到正在传输的数据量，发送方仍被允许发送的数据量。实际上等于第3类（未发送接收方ready）的大小。


确认处理和窗口缩放：
1、发送可用窗口内字节，可用窗口缩小为空。
2、接收方会发送成功接收后的最长字节数。
3、发送方收到这个最长字节数后，向后滑动对应字节数，可用窗口扩大对应字节数。

### TCP重传机制
TCP片段重传计时器 + 重传队列
简单的方法：每次发送一个片段，就开启一个重传计时器，如果超时未确认，就重传片段。
高效的方法：
	发送片段时，将副本放置于重传队列中，计时器开始。
	收到确认消息则重队列移除该片段。
	如果重传超时，则rest计时
	器再次重传，直到超过限定次数，并判断出现故障终止连接。

Q 要知道重传是基于片段的，而TCP确认消息是基于序列号累计的，怎样判断一个片段被完全确认呢？
A 

Q 如何判断一个片段为丢失片段？

Q 重传计时器，如果时间太短容易发生过量重传，太长容易导致传输效率低下。
A 如何设置呢：自适应动态调整。

### TCP ACK与重传机制
- 仅重传超时的片段
- 重传超时+后面的所有片段
- 快速重传机制
TCP引入了一种叫Fast Retransmit 的算法，不以时间驱动，而以数据驱动重传。
也就是说，如果包没有连续到达，就ack最后那个可能被丢了的包，
如果发送方连续收到3次相同的ack，就重传。
Fast Retransmit的好处是不用等timeout了再重传。

- SACK，需要在握手阶段，双方协商同时支持此机制：
问题的关键在于无法确认非连续片段。
解决方式是对TCP滑动窗口算法进行扩展，添加允许设备分别确认非连续片段的功能。
这一功能称为选择确认（selective acknowledgment, SACK）。

Duplicate-SACK	
Duplicate SACK又称D-SACK，其主要使用了SACK来告诉发送方有哪些数据被重复接收了
可见，引入了D-SACK，有这么几个好处：

1）可以让发送方知道，是发出去的包丢了，还是回来的ACK包丢了。
2）是不是自己的timeout太小了，导致重传。
3）网络上出现了先发的包后到的情况（又称reordering）
4）网络上是不是把我的数据包给复制了。
知道这些东西可以很好得帮助TCP了解网络情况，从而可以更好的做网络上的流控。
Linux下的tcp_dsack参数用于开启这个功能（Linux 2.4后默认打开）


### TCP RTT与Timeout超时重传
TCP重传机制中Timeout的设置非常重要：
设长了，重发就慢，丢了老半天才重发，没有效率，性能差；
设短了，会导致可能并没有丢就重发。于是重发的就快，会增加网络拥塞，导致更多的超时，更多的超时导致更多的重发。

为了动态设置Timeout，引入RTT评估当前传输速率: round trip time
再按RTT设置重传时间RTO(Retransmission Timeout)

- RFC793定义的经典算法：Soothed RTT
- Karn-Partride算法：忽略重传的case，不对重传RTT做采样
- Jacobson-Karels算法：前两种都是“加权平均移动”，很难处理抖动大的情况

TCP ACK Timeout: 重传
TCP Premature Timeout: 超时设置太短导致超时太快，ACK会返回最大的sequence number
TCP Cumulative ACK: ACK会返回最大的sequence number

### TCP窗口调整与流量控制
窗口的本质：
代表设备对于特定链接接收缓冲区大小，即窗口大小代表了一次能够对端处理多少数据，之后再传递给应用层处理。


### Zero Window
Q 如果数据接收方处理太慢，那么窗口大小逐渐变成0，那么接收方一会window size可用了，怎么通知发送端呢？
A Zero Window Probe(ZWP)，发送端窗口在变成0后，会发送ZWP包给接收方，让接收方ack他的window尺寸，
ZWP包会定期发送几次（一般是3次），每次大约30-60秒（不同实现可能不一样），如果最后还是0的话，有的TCP实现就会发送RST把链接断了。

### Silly Window Syndrome
小包问题
MTU 最大传输单元，对以太网Ethernet来说，MTU是1500字节，除去TCP+IP Header的40字节，真正传输有效数据是1460，
这就是所谓的MSS(Max Segment Size)。
https://en.wikipedia.org/wiki/Maximum_transmission_unit

对于Reciever: David D. Clark's 方案 ，如果接收数据导致window size小于某个值，可以直接ack(0)返回sender.
等到reciever处理掉了一些数据导致window size大于等于MSS，或者，reciever buffer有一半为空，就可以把window 打开让sender数据过来。

对于Sender: Nagle's Algorithm，这个算法的思路也是延时处理，有两个主要条件：
1）等到Window size >= MSS 或者 DataSize >= MSS
2) 收到之前发送数据的ack回包，他才会发数据，否则就是在攒数据。

另外，Nagle算法默认是打开的，所以，对于一些需要小包场景的程序——比如像telnet或ssh这样的交互性比较强的程序，
你需要关闭这个算法。你可以在Socket设置TCP_NODELAY选项来关闭这个算法


## TCP拥塞处理-Congestion Handling
https://www.youtube.com/watch?v=cPLDaypKQkU
https://en.wikipedia.org/wiki/TCP_congestion_control#Congestion_window

In TCP, <b>the congestion window</b> is one of the factors that determines the number of bytes
that can be sentout at any time.
The congestion window is maintained by the sender and is a means of stopping a link
between the sender and the reciever from becoming overloaded with too much traffic.

This shoud not be confused with the sliding window maintained by the reciever which
exists to prevent the <i>reciever</i> from becoming overloaded. The congestion window is calculated by estimating how much congestion there is on the link.

When a connection is set up, the congestion window, a value maintained independently at each host, is set to a small multiple of the MSS allowed on that connection. Further variance in the congestion window is dictated by an AIMD approach. 
This means that if all segments are recieved the acknowledgements reach the sender on time, some constant is added to the window size.
When the window reaches sshresh, the congestionn window increases linearly at the rate of 1/(congestion window) segment on each new acknowlegement recieved. 
The window keeps growing until a timeout occurs.

On timeout:
1. Congestion is reset to 1 MSS.
2. sshtresh is set to half the congestion window size before the timeout.
3. slow start is intiated.



如果网络延迟突然增加，TCP只是无脑重传，会导致网络负担更重，恶性循环下去，会形成“网络风暴”，最后拖垮整个网络。
所以，TCP不能忽略网络上发送的事情，不应该无脑重发数据，对网络造成更大的伤害。

对此，TCP的设计理念是：
<font color='red'>TCP不是一个自私的协议，当拥塞发生时，要做自我牺牲，
就像交通堵塞一样，每个车都应该把路让出来，而不要再去抢路了。</font>

- 1988年，TCP-Tahoe 提出了1）慢启动，2）拥塞避免，3）拥塞发生时的快速重传
- 1990年，TCP Reno 在Tahoe的基础上增加了4）快速恢复

控制拥塞主要是四个算法：慢启动、拥塞避免、拥塞发生、快速恢复。
形成的波形图：指数上升（慢启动）+ 线性上升（拥塞避免） + 断崖下跌（拥塞发生）
最终的效果是：sender不断向上探测，遇到丢包重传、超时重传对发送速度进行惩罚，从而达到最佳发送速率。
其实拥塞控制有2种思路：一种是方法是基于丢包来降低发送速率，一种是主动探测(google BBR)。

AIMD和式增加，积式减少
additive-increase/multiplicative-decrease，AIMD，这里简称“线增积减”.
是一种反馈控制算法，其包含了对拥塞窗口线性增加，和当发生拥塞时对窗口积式减少。多个使用AIMD控制的TCP流最终会收敛到对线路的等量竞争使用。

### 发送速率
rate = cwnd/RTT
cwnd: 拥塞窗口Congestion Window，每次ACK后，发送bytes数，根据网络拥塞情况自适应调整。
发送太快则导致网络拥塞，发送太慢则导致低效，网络情况不断在变，如何动态调整发送速率？
不断向上探测，如果发生丢包则降低速率。

https://homes.cs.washington.edu/~tom/pubs/pacing.pdf


### 慢启动算法 Slow start
LastByteSent - LastByteAcked <= cwnd
cwnd: congestion window
ssthreshold: slow start threshold

### 拥塞避免算法 Congestion avoidance
前面说过，还有一个ssthresh（slow start threshold），是一个上限，当cwnd >= ssthresh时，就会进入“拥塞避免算法”。一般来说ssthresh的值是65535，单位是字节，当cwnd达到这个值时后，算法如下：

1）收到一个ACK时，cwnd = cwnd + 1/cwnd
如果一个cwnd全部被ACK，最后效果相当于cwnd = cwnd + 1

2）当每过一个RTT时，cwnd = cwnd + 1

这样就可以避免增长过快导致网络拥塞，慢慢的增加调整到网络的最佳值。很明显，是一个线性上升的算法。

### 拥塞发生时的算法 Congestion recovery
- case1 RTO超时，重传数据包，TCP认为这种情况太糟糕，反应也很强烈：
    ssthreshold=cwnd/2、cwnd重置为1、进入慢启动过程

- case2 丢包，收到3个duplicate ACK(fast retransmit算法)开启重传，不用等到RTO超时。
    TCP Tahoe的实现和RTO超时一样
    TCP Reno的实现是：cwnd折半，sshthresh = cwnd，进入快速恢复算法Fast Recovery

### 快速恢复算法Fast Recovery


### 向前确认算法 FACK
FACK: Forward Acknowlegement算法，基于SACK(selective acknowlegement).
SACK可以让发送端这边再重传过程中，把哪些丢掉的包重传，而不是一个个的传，但是如果重传的包数据表多的话，又会导致本来很忙的网络更忙了。所以，FACK用来做重传过程中的拥塞流控。


## 其它拥塞控制算法简介


总结一下：
TCP为了避免reciever处理不过来，使用滑动窗口+ack机制。
TCP重传机制：1、超时重传机制；2、快速重传机制（3个重复ACK）
SACK

## 半连接
1）发送端发送了SYN报文
2）接收端收到后，返回SYN+ACK给发送端，并将socket放到sync queue
3）发送端收到报文后，发送ACK给发送端，发送端将socket移入accept queue
