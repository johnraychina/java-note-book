
https://lmax-exchange.github.io/disruptor/
https://tech.meituan.com/2016/11/18/disruptor.html


## 介绍
学习新东西的最好方式就是类比现有的东西。
Disruptor可以类比为Java的BlockingQueue。
为了将数据（消息、事件）在一个进程中的多个线程之间高效移动，做了一些改进：
- 按consumer的依赖图，事件广播给多个consumer。
- 给事件预分配内存
- （可选）无锁

## 核心概念：

- RingBuffer 环形缓冲队列
用数组而非链表，规避垃圾回收问题。


- Sequence 序列
和Disruptor本身一样，每个消费者（EventProcessor)都维护了一个Sequence。
大部分的并发代码都依赖于Sequence。
它类似于AtomicLong，但是解决了false sharing问题

- Sequencer 排序器，
这是Disruptor真正的核心。
（single producer, multi producer)2个实现的并发算法保证让数据在生产者和消费者之间快速、正确地流动。


- Sequence Barrier 
它由Sequencer生成   
包含了指向发布Sequence和消费Sequence   
它包含是否有事件对消费者可用的逻辑   

- Event

- EventProcessor
它是事件处理的event loop
是消费者Sequence的所有者


- EventHandler

- Producer

**Multicast Even**
Disruptor和普通队列的最大差别就在于广播事件**


**Consumer Dependency Graph**

**Event Preallocation**
为了降低延迟，做预分配。

**Optionally Lock-free**
无锁化处理

使用内存屏障 和 CAS操作

内存复制


## 如何使用？

实现生产者：new EventFactory
实现消费者：EventHandler
组装到Disruptor：new Disruptor









