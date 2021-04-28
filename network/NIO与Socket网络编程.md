

InfoQ Java NIO：https://xie.infoq.cn/article/1f44643161e1666b6a30b85e7
美团Java NIO解析：https://tech.meituan.com/2016/11/04/nio.html
C10K问题：http://www.kegel.com/c10k.html#frameworks
cnblog：https://www.cnblogs.com/straight/articles/7660889.html
简书：https://www.jianshu.com/p/2b71ea919d49
https://zhuanlan.zhihu.com/p/34371409
https://zhuanlan.zhihu.com/p/23114695

## C10K 问题
server端如果用BIO模型，则必须每个连接分配一个线程处理。
开一个线程按默认配置要1M的内存，并且太多线程切换起来也会消耗过多CPU time， 非常耗资源。
上千个线程几乎就要耗尽系统资源了，所以BIO模型无法解决上万的连接问题。
要解决这个问题，要从操作系统Socket来看。

## C10M 问题 todo
youtube: C10M Defending The Internet At Scale, Robert David Graham


## 操作系统Socket
操作系统把对网络的操作封装为socket, 应用通过socket系统调用完成网络操作，所以有一些书名叫“socket编程”。
unix/linux中一切资源皆为File，都可以用“open –> write/read –> close”模式来操作。
所以socket连接对应着操作系统的文件描述符(file descriptor).

server socket: 
    int socket(int domain, int type, int protocol) 创建socket
    bind 绑定(ip)地址
    listen 监听，对于SOCK_STREAM类型的socket，会指定backlog：连接数队列上限
    accept 等待获取client的连接
    send/recv/read/write 收发消息
client socket: 
    int socket(int domain, int type, int protocol) 创建socket
    connect 向server发送连接请求
    send/recv/read/write 收发消息


### 操作系统提供的IO模型：
- recvfrom 支持阻塞式IO、非阻塞式

IO多路复用
- select 同步的IO多路复用，对IO file descriptor(fd)进行遍历检查：是否读就绪，写就绪 或者异常。select中的fd是用数组实现的，有长度限制。
- poll 和select差不多，同步的IO多路复用，主要是将fd数组改为链表，解决了长度问题。     
- epoll 事件驱动，红黑树的结构保存fd句柄，以支持快速的查找、插入、删除

- async io 异步io
- signal 信号驱动

socket IO实际分2个阶段：
- 等待数据(检查是否有数据)
- 应用程序调用socket recv -> 操作系统把数据从内核拷贝到用户空间
socket IO模型中的<b>阻塞</b>是指第一阶段等待数据，第二阶段很快而且是必须等待的（除非用直接内存访问DirectBuffer)

select/poll/epoll模型可以将第一阶段【等待数据】 变为 【轮询检查是否有数据，立即返回读/写就绪的fd】
异步则是好莱坞原则（don't call me, I'll call you)，应用把<event type, event handler>注册到socket, 当有你关心的事件来的时候，操作系统会来call事件处理器。

#### epoll介绍
https://zhuanlan.zhihu.com/p/34371409
IO模型不断在改进，从select到poll再到epoll，epoll目前已经是主流的编程模型。

- 最大连接数：
epoll 没有最大并发连接的限制，上限是最大可以打开文件的数目

- socket事件遍历效率：
epoll对于句柄事件的选择不是遍历的，是事件响应的，就是句柄上事件来就马上选择出来，不需要遍历整个句柄链表，因此效率非常高。
内核将句柄用红黑树保存的，IO效率不随FD数目增加而线性下降。

- 内存拷贝：
select让内核把 FD 消息通知给用户空间的时候使用了内存拷贝的方式，开销较大。
但是Epoll 在这点上使用了共享内存的方式，这个内存拷贝也省略了。

#### epoll状态

EPOLLIN ： 可读
EPOLLOUT： 可写
EPOLLPRI： 紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR： 错误；
EPOLLHUP： 挂断；
EPOLLET： 将 EPOLL设为边缘触发(Edge Triggered)模式（默认为水平触发），这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT： 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

**libevent 采用水平触发， nginx 采用边沿触发.**

#### 使用epoll:
使用 epoll_create 在内核中创建一个上下文；
使用 epoll_ctl 向/从上下文添加/移除文件描述符；
使用 epoll_wait 等待上下文中的事件。

```c
 1 for( ; ; )
 2     {
 3         nfds = epoll_wait(epfd,events,20,500);
 4         for(i=0;i<nfds;++i)
 5         {
 6             if(events[i].data.fd==listenfd) //如果是主socket的事件，则表示有新的连接
 7             {
 8                 connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen); //accept这个连接
 9                 ev.data.fd=connfd;
10                 ev.events=EPOLLIN|EPOLLET;
11                 epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //将新的fd添加到epoll的监听队列中
12             }
13             else if( events[i].events&EPOLLIN ) //接收到数据，读socket
14             {
15 
16             if ( (sockfd = events[i].data.fd) < 0) continue;
17                 n = read(sockfd, line, MAXLINE)) < 0    //读
18                 ev.data.ptr = md;     //md为自定义类型，添加数据
19                 ev.events=EPOLLOUT|EPOLLET;
20                 epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识符，等待下一个循环时发送数据，异步处理的精髓
21             }
22             else if(events[i].events&EPOLLOUT) //有数据待发送，写socket
23             {
24                 struct myepoll_data* md = (myepoll_data*)events[i].data.ptr;    //取数据
25                 sockfd = md->fd;
26                 send( sockfd, md->ptr, strlen((char*)md->ptr), 0 );        //发送数据
27                 ev.data.fd=sockfd;
28                 ev.events=EPOLLIN|EPOLLET;
29                 epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改标识符，等待下一个循环时接收数据
30             }
31             else
32             {
33                 //其他情况的处理
34             }
35         }
36     }


```

#### epoll wait 事件触发：水平触发 VS 边沿触发
https://blog.csdn.net/NK_test/article/details/49331717
https://www.cnblogs.com/panfeng412/articles/2229095.html

1、level-triggered (LT) 水平触发：
完全靠Linux-kernel-epoll驱动，应用程序只需要处理从epoll_wait返回的fds， 这些fds我们认为它们处于就绪状态。
此时epoll可以认为是更快速的poll。

2、edge-triggered (ET) 边沿触发：
此模式下，系统仅仅通知应用程序哪些fds变成了就绪状态，一旦fd变成就绪状态，epoll将不再关注这个fd的任何状态信息(从epoll队列移除), 

直到应用程序通过读写操作（非阻塞）触发EAGAIN状态，epoll认为这个fd又变为空闲状态，那么epoll又重新关注这个fd的状态变化(重新加入epoll队列)。 

随着epoll_wait的返回，队列中的fds是在减少的，所以在大并发的系统中，EPOLLET更有优势，但是对程序员的要求也更高，因为有可能会出现数据读取不完整的问题：
   假设现在对方发送了2k的数据，而我们先读取了1k，然后这时调用了epoll_wait：
    如果是边沿触发ET，那么这个fd变成就绪状态就会从epoll 队列移除，则epoll_wait 会一直阻塞，忽略尚未读取的1k数据; 
    而如果是水平触发LT，那么epoll_wait 还会检测到可读事件而返回，我们可以继续读取剩下的1k 数据。

   因此总结来说: LT模式可能触发的次数更多, 一旦触发的次数多, 也就意味着效率会下降; 
   但这样也不能就说LT模式就比ET模式效率更低, 因为ET的使用对编程人员提出了更高更精细的要求,一旦使用者编程水平不够, 那ET模式还不如LT模式;


# Java NIO
Java NIO(应该理解为java New IO，其中包含blocking和non-blocking IO)

- Selector 是实现线程多路复用的核心，原理是非阻塞地遍历多个socket channel，得知是否就绪。
具体实现：PollSelectorImpl, KQueueSelectorImpl
```java
    Selector.open();
```

- SocketChannel: A selectable channel for stream-oriented connecting sockets.
 面向流的socket连接
```java
    ServerSocketChannel.open();
```

- Socket 操作系统提供的网络操作接口
A socket is an endpoint for communication between two machines.

- Buffer 缓冲，分为blocking 和 nonblocking


# Netty做了什么？
基于Java NIO，实现了更好的编程模型，简化socket的繁琐操作。
EventLoop事件机制，异步编程。

