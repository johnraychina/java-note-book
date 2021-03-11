
https://en.wikipedia.org/wiki/Transmission_Control_Protocol
https://www.cnblogs.com/xiaolincoding/p/12638546.html

> 什么是TCP？

TCP 是面向连接的、可靠的、基于字节流的传输层通信协议。

面向连接：一定是「一对一」才能连接，不能像 UDP 协议可以一个主机同时向多个主机发送消息，也就是一对多是无法做到的；

可靠的：无论的网络链路中出现了怎样的链路变化，TCP 都可以保证一个报文一定能够到达接收端；

字节流：消息是「没有边界」的，所以无论我们消息有多大都可以进行传输。并且消息是「有序的」，当「前一个」消息没有收到的时候，即使它先收到了后面的字节，那么也不能扔给应用层去处理，同时对「重复」的报文会自动丢弃。


> 什么是 TCP 连接？

Connections: The reliability and flow control mechanisms described above require that TCPs initialize and maintain certain status information for each data stream. The combination of this information, including sockets, sequence numbers, and window sizes, is called a connection.

简单来说就是，用于保证可靠性和流量控制维护的某些状态信息，这些信息的组合，包括Socket、序列号和窗口大小称为连接。
[Linux每个TCP连接消耗内存大概是3.15KB](https://zhuanlan.zhihu.com/p/25241630)


所以我们可以知道，建立一个 TCP 连接是需要客户端与服务器端达成上述三个信息的共识。

- Socket：由 IP 地址和端口号组成
- 序列号：用来解决乱序、丢包等问题
- 窗口大小：用来做流量控制


> 如何唯一确定一个 TCP 连接呢？

四元组：sourceIp + sourcePort + destIp + destPort


# IPv4 header(20 Bytes)
source ip: 4 Bytes
dest ip: 4 Bytes


# TCP Header(20~60 Bytes)
- source port: 2 Bytes
- dest port: 2 Bytes
- seq no: 4 Bytes
- ack no: 4 Bytes
- data offset: 4 bits
- reserved: 3 bits
- flags: 9 bits(NS/CWR/ECE/URG/ACK/PSH/RST/SYN/FIN)
- window size: 2 Bytes
- check sum: 2 Bytes
- Urgent pointer: 2 Bytes
- options: 0-40 bytes
- padding: ensure that  TCP header ends, and data begins, on a 32-bits boundary.

seq no 序列号：每次发送数据包时，累加这个数字，用来解决网络包乱序的问题。

ack no 确认应答号：接收到数据后，接收端会回复下次【期望接收包序号】，发送端可以以此作为发送依据，用来解决丢包问题。

data offset数据偏移量: 衡量tcp header大小（以32bits word为单位），header大小是[5~15]words，即[20~60]Bytes, 允许最多40 Bytes的option部分.



flags控制位：
ACK：该位为 1 时，「确认应答」的字段变为有效，TCP 规定除了最初建立连接时的 SYN 包之外该位必须设置为 1 。
RST：该位为 1 时，表示 TCP 连接中出现异常必须强制断开连接。
SYN：该位为 1 时，表示希望建立连接，并在其「序列号」的字段进行序列号初始值的设定。
FIN：该位为 1 时，表示今后不会再有数据发送，希望断开连接。当通信结束希望断开连接时，通信双方的主机之间就可以相互交换 FIN 位为 1 的 TCP 段。


# UDP header
UDP不保证可靠传输，没有那么多复杂的状态控制机制，利用IP提供面向【无连接】的通信服务。

所以UDP协议非常简单，UDP header 只有8个字节：
- sourcePort(2Bytes) 
- destPort(2Bytes) 
- length(2Bytes) 
- checksum(2Bytes)


# TCP 和 UDP 区别：

1. 连接

TCP 是面向连接的传输层协议，传输数据前先要建立连接。
UDP 是不需要连接，即刻传输数据。
2. 服务对象

TCP 是一对一的两点服务，即一条连接只有两个端点。
UDP 支持一对一、一对多、多对多的交互通信
3. 可靠性

TCP 是可靠交付数据的，数据可以无差错、不丢失、不重复、按需到达。
UDP 是尽最大努力交付，不保证可靠交付数据。
4. 拥塞控制、流量控制

TCP 有拥塞控制和流量控制机制，保证数据传输的安全性。
UDP 则没有，即使网络非常拥堵了，也不会影响 UDP 的发送速率。
5. 首部开销

TCP 首部长度较长，会有一定的开销，首部在没有使用「选项」字段时是 20 个字节，如果使用了「选项」字段则会变长的。
UDP 首部只有 8 个字节，并且是固定不变的，开销较小。
6. 传输方式

TCP 是流式传输，没有边界，但保证顺序和可靠。
UDP 是一个包一个包的发送，是有边界的，但可能会丢包和乱序。
7. 分片不同

TCP 的数据大小如果大于 MSS 大小，则会在传输层进行分片，目标主机收到后，也同样在传输层组装 TCP 数据包，如果中途丢失了一个分片，只需要传输丢失的这个分片。
UDP 的数据大小如果大于 MTU 大小，则会在 IP 层进行分片，目标主机收到后，在 IP 层组装完数据，接着再传给传输层，但是如果中途丢了一个分片，则就需要重传所有的数据包，这样传输效率非常差，所以通常 UDP 的报文应该小于 MTU。

# TCP 和 UDP 应用场景
由于 TCP 是面向连接，能保证数据的可靠性交付，因此经常用于：
- FTP 文件传输
- HTTP / HTTPS

由于 UDP 面向无连接，它可以随时发送数据，再加上UDP本身的处理既简单又高效，因此经常用于：
- 包总量较少的通信，如 DNS 、SNMP 等
- 视频、音频等多媒体通信
- 广播通信


# TCP 建立连接（三次握手）

client--(SYN, seq_no=client_isn)--> server
client: SYNC_SENT
server: LISTNEN -> SYNC_RCVD

client<--(SYN+ACK, seq_no=server_isn, ack_no=client_isn+1)--server
client: SYNC_SENT -> ESTABLISHED
server: SYNC_RCVD

client--(ACK, seq_no=server_isn+1)--server
client: ESTABLISHED
server: SYNC_RCVD -> ESTABLISHED

从上面的过程可以发现第三次握手是可以携带数据的，前两次握手是不可以携带数据的，这也是面试常问的题。

> 如何在 Linux 系统中查看 TCP 状态？

netstat -napt


> 为什么是三次握手？不是两次、四次？

建立可靠连接最小的通信次数就是三次：序列号同步

- 三次握手的首要原因是为了防止旧的重复连接初始化造成混乱。
如果是历史连接（序列号过期或超时），则第三次握手发送的报文是 RST 报文，以此中止历史连接；
如果不是历史连接，则第三次发送的报文是 ACK 报文，通信双方就会成功建立连接；
RFC 793: The principle reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion.

- 同步双方初始序列号
一来一回，才能确保双方的初始序列号能被可靠的同步。
四次握手其实也能够可靠的同步双方的初始化序号，但由于第二步和第三步可以优化成一步，所以就成了「三次握手」。


- 避免资源浪费
服务端建立连接采用1阶段确策略：第一个包到达server直接建立连接，如果中间因为网络问题超时，第二个包未及时到达client，server是不知道的，无法释放连接资源。
服务端建立连接采用2阶段确认策略：如果中间因为网络问题超时，第二个包未及时到达client，或者第三个包未及时到达server，server可以释放连接资源。



> 初始序列号 ISN 是如何随机产生的？

起始 ISN 是基于时钟的，每 4 毫秒 + 1，转一圈要 4.55 个小时。

RFC1948 中提出了一个较好的初始化序列号 ISN 随机生成算法。

ISN = M + F (localhost, localport, remotehost, remoteport)

- M 是一个计时器，这个计时器每隔 4 毫秒加 1。
- F 是一个 Hash 算法，根据源 IP、目的 IP、源端口、目的端口生成一个随机数值。要保证 Hash 算法不能被外部轻易推算得出，用 MD5 算法是一个比较好的选择。

> 既然 IP 层会分片，为什么 TCP 层还需要 MSS 呢？

MTU 与 MSS
MTU是IP层做的分片：一个网络包的最大长度，以太网中一般为 1500 字节；
MSS是TCP层做的分片：除去 IP 和 TCP 头部之后，一个网络包所能容纳的 TCP 数据的最大长度；

因为 IP 层本身没有超时重传机制，它由传输层的 TCP 来负责超时和重传。
所以握手时要协商MSS，保证加上TCP header 和 IP header不会超过MTU。


因为 IP 层本身没有超时重传机制，那么当如果一个 IP 分片丢失，整个 IP 报文的所有分片都得重传。
由于 TCP可靠传输机制，经过 TCP 层分片后，如果一个 TCP 分片丢失后，进行重发时也是以 MSS 为单位，而不用重传所有的分片，大大增加了重传的效率。

> 什么是 SYN 攻击？如何避免 SYN 攻击？

client一直请求连接，但是不发ACK，导致占满server的SYN接收队列。

- 解法一： 通过修改 Linux 内核参数，控制队列大小和当队列满时应做什么处理。
当网卡接收数据包的速度大于内核处理的速度时，会有一个队列保存这些数据包：net.core.netdev_max_backlog
SYN_RCVD 状态连接的最大个数：net.ipv4.tcp_max_syn_backlog
超出处理能时，对新的 SYN 直接回报 RST，丢弃连接：net.ipv4.tcp_abort_on_overflow

- 解法二：设置net.ipv4.tcp_syncookies = 1
如果应用程序过慢时，就会导致「 Accept 队列」被占满。
如果不断受到 SYN 攻击，就会导致「 SYN 队列」被占满。
    - 当 「 SYN 队列」满之后，后续服务器收到 SYN 包，不进入「 SYN 队列」；
    - 计算出一个 cookie 值，再以 SYN + ACK 中的「序列号」返回客户端，
    - 服务端接收到客户端的应答报文时，服务器会检查这个 ACK 包的合法性。如果合法，直接放入到「 Accept 队列」。
    - 最后应用通过调用 accpet() socket 接口，从「 Accept 队列」取出的连接。


# TCP 消息接收
TCP是基于流，按序号传输机制。

流：接收队列，按seq_no排序

消息：对流中的报文组合拆分：按分隔符(delimiter-based) 或者长度(length-based)，参见Netty粘包、拆包。


# TCP 连接断开

> 为什么挥手需要四次？

client --(client FIN)--> server
client <--(server ACK)-- server
服务端数据处理中
client <--(server FIN)-- server
client --(client ACK) --> client
client等待2MSL后关闭连接

服务端通常需要等待完成数据的发送和处理，所以服务端的 ACK 和 FIN 一般都会分开发送，从而比三次握手导致多了一次。

> 为什么 TIME_WAIT 等待的时间是 2MSL？

MSL 是 Maximum Segment Lifetime，报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。因为 TCP 报文基于是 IP 协议的，而 IP 头中有一个 TTL 字段，是 IP 数据报可以经过的最大路由数，每经过一个处理他的路由器此值就减 1，当此值为 0 则数据报将被丢弃，同时发送 ICMP 报文通知源主机。

MSL 与 TTL 的区别： MSL 的单位是时间，而 TTL 是经过路由跳数。所以 MSL 应该要大于等于 TTL 消耗为 0 的时间，以确保报文已被自然消亡。

TIME_WAIT 等待 2 倍的 MSL，比较合理的解释是： 网络中可能存在来自发送方的数据包，当这些发送方的数据包被接收方处理后又会向对方发送响应，所以一来一回需要等待 2 倍的时间。

比如如果被动关闭方没有收到断开连接的最后的 ACK 报文，就会触发超时重发 Fin 报文，另一方接收到 FIN 后，会重发 ACK 给被动关闭方， 一来一去正好 2 个 MSL。

2MSL 的时间是从客户端接收到 FIN 后发送 ACK 开始计时的。如果在 TIME-WAIT 时间内，因为客户端的 ACK 没有传输到服务端，客户端又接收到了服务端重发的 FIN 报文，那么 2MSL 时间将重新计时。

在 Linux 系统里 2MSL 默认是 60 秒，那么一个 MSL 也就是 30 秒。Linux 系统停留在 TIME_WAIT 的时间为固定的 60 秒。

> 为什么需要 TIME_WAIT 状态？
这里一点需要注意是：主动关闭连接的，才有 TIME_WAIT 状态。

- 原因一：防止端口复用时，收到旧连接的数据包
- 原因二：保证连接正确关闭
RFC 793: TIME-WAIT - represents waiting for enough time to pass to be sure the remote TCP received the acknowledgment of its connection termination request.
如果 TIME-WAIT 等待足够长的情况就会遇到两种情况：
    - 服务端正常收到四次挥手的最后一个 ACK 报文，则服务端正常关闭连接。
    - 服务端没有收到四次挥手的最后一个 ACK 报文时，则会重发 FIN 关闭连接报文并等待新的 ACK 报文。

> TIME_WAIT 过多有什么危害？
- 第一是内存资源占用；
- 第二是对端口资源的占用，一个 TCP 连接至少消耗一个本地端口；

> 如何优化 TIME_WAIT？
- 打开 net.ipv4.tcp_tw_reuse 和 net.ipv4.tcp_timestamps 选项；
- net.ipv4.tcp_max_tw_buckets
- 程序中使用 SO_LINGER ，应用强制使用 RST 关闭。

> 如果已经建立了连接，但是客户端突然出现故障了怎么办？
TCP 有一个机制是保活机制

> listen 时候参数 backlog 的意义？

在早期 Linux 内核 backlog 是 SYN 队列大小，也就是未完成的队列大小。

在 Linux 内核 2.2 之后，backlog 变成 accept 队列，也就是已完成连接建立的队列长度，所以现在通常认为 backlog 是 accept 队列。

但是上限值是内核参数 somaxconn 的大小，也就说 accpet 队列长度 = min(backlog, somaxconn)。

> accept 发生在三次握手的哪一步？
accept开始于server接收到第一个包，等第三个包到了之后会返回。

> 客户端调用 close 了，连接是断开的流程是什么？

- 客户端调用 close，表明客户端没有数据需要发送了，则此时会向服务端发送 FIN 报文，进入 FIN_WAIT_1 状态；

- 服务端接收到了 FIN 报文，TCP 协议栈会为 FIN 包插入一个文件结束符 EOF 到接收缓冲区中，应用程序可以通过 read 调用来感知这个 FIN 包。这个 EOF 会被放在已排队等候的其他已接收的数据之后，这就意味着服务端需要处理这种异常情况，因为 EOF 表示在该连接上再无额外数据到达。此时，服务端进入 CLOSE_WAIT 状态；

- 接着，当处理完数据后，自然就会读到 EOF，于是也调用 close 关闭它的套接字，这会使得客户端会发出一个 FIN 包，之后处于 LAST_ACK 状态；
- 客户端接收到服务端的 FIN 包，并发送 ACK 确认包给服务端，此时客户端将进入 TIME_WAIT 状态；
- 服务端收到 ACK 确认包后，就进入了最后的 CLOSE 状态；
- 客户端经过 2MSL 时间之后，也进入 CLOSE 状态；

# 巨人的肩膀
[1] 趣谈网络协议专栏.刘超.极客时间.

[2] 网络编程实战专栏.盛延敏.极客时间.

[3] 计算机网络-自顶向下方法.陈鸣 译.机械工业出版社

[4] TCP/IP详解 卷1：协议.范建华 译.机械工业出版社

[5] 图解TCP/IP.竹下隆史.人民邮电出版社

[6] https://www.rfc-editor.org/rfc/rfc793.html

[7] https://draveness.me/whys-the-design-tcp-three-way-handshake

[8] https://draveness.me/whys-the-design-tcp-time-wait/