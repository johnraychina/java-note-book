# Java 并发工具看作者 Doug Lea
http://ifeve.com/doug-lea/
http://gee.cs.oswego.edu/


# atomic工具
基于CAS, getAndAdd
CPU cache padding缓存填充
分散计数思想

https://www.jianshu.com/p/30d328e9353b

# 锁工具
LockSupport park/unpark 调用native方法实现线程阻塞唤醒
ReentrantLock, ReentrantReadWriteLock 锁
CountdownLatch 飞机等人模型
Semaphore 有限资源循环利用
CyclicBarrier 游戏人等人模型

- Semaphore
基于ReentrantLock基于AQS基于CAS
共享锁，有足够的permit许可时，setHeadAndPropagate会传播给队列中SHARED共享的线程


- CountDownLatch
基于ReentrantLock基于AQS基于CAS，共享模式。
注意事项：主线程先new CountDownLatch, 子线程必须先countDown，然后主线程再做await
原理是：主线程new完CountDownLatch 里面的state=n，然后子线程countDown->tryAcquireShared

- CyclicBarrier
基于ReentrantLock基于AQS基于CAS

- Exchanger
基于 ThreadLocal + for循环{ CAS }
http://www.cs.rochester.edu/research/synchronization/pseudocode/duals.html

- ReentrantReadWriteLock

1.7新工具：
- Phaser

1.8新工具
- StampedLock
for + CAS

# ReentrantLock 与 Condition
Condition使用场景：有界缓冲，一个线程写满了就等，一个线程读空了就等，ArrayBlockingQueue与object monitor不同的是：
1.提供了一种顺序明确的通知机制
2.通知时，不需要获得锁

```java
   class BoundedBuffer {
     final Lock lock = new ReentrantLock();
     final Condition notFull  = lock.newCondition(); 
     final Condition notEmpty = lock.newCondition(); 
  
     final Object[] items = new Object[100];
     int putptr, takeptr, count;
  
     public void put(Object x) throws InterruptedException {
       lock.lock();
       try {
         while (count == items.length)
           notFull.await();
         items[putptr] = x;
         if (++putptr == items.length) putptr = 0;
         ++count;
         notEmpty.signal();
       } finally {
         lock.unlock();
       }
     }
  
     public Object take() throws InterruptedException {
       lock.lock();
       try {
         while (count == 0)
           notEmpty.await();
         Object x = items[takeptr];
         if (++takeptr == items.length) takeptr = 0;
         --count;
         notFull.signal();
         return x;
       } finally {
         lock.unlock();
       }
     }
   }
```
实现时需要注意的是：考虑于操作系统的spurious wakups机制，condition.await()最好放循环中：
while(条件不满足) {
    condition.await()
}

# FutureTask


# 底层：基于AQS(AbstractQueuedSynchronizer)实现
https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html
https://juejin.im/post/6844903895622221832



## 总结
AQS底层：基于FIFO双端队列 + CAS volatile state + LockSupport.park/unpark
lock -> AQS.acquire -> addWaiter加到队列tail -> AQS.acquireQueued { for循环tryAcquire + LockSupport.park }
unlock -> AQS.release -> 取next，若无则从tail遍历取等待的线程做 LockSupport.park

## 详细设计--数据结构是关键
**看java doc才是精髓**
AQS:代表多线程访问的某个资源
AQS.state: 资源同步锁定状态，为0表示未锁定（可用)， 通过volatile保证可见性，通过CAS保证原子操作
AQS.exclusiveOwnerThread: 代表当前占用资源的线程
AQS.head-tail: sync queue加锁线程等待队列，head是个dummy header, head.next代表第一个节点（注意第一个节点不一定是exclusiveOwnerThread哦）
Node: 代表请求资源的线程的
Node.status状态: 
  CANCELLED(1) 本节点线程对AQS的加锁超时或者被中断，后续处理需要忽略。
  SIGNAL(-1) 本节点线程对AQS加锁失败，被park阻塞，需要信号来恢复
  CONDITION(-2) 本节点线程对AQS的加锁条件未满足，放在condition queue中，等到合适时机会被设置为0然后转移到sync queue中
  PROPAGATE(-3) 共享模式下的释放操作应该被传播到其他节点。该状态值在doReleaseShared方法中被设置的。
            indicate the next acquireShared should unconditionally propagate
  0 None of the above
Node.SHARED/EXCLUSIVE 资源加锁模式: 共享/独占
AQS.Node.nextWaiter EXCLUSIVE/SHARED 
AQS.ConditionObject.Node.nextWaiter 等待条件的下个节点（由于condition queue是独占访问的）

释放通知：release释放锁的时候，会发出通知，等待队列中的线程可以抢占。

### condition条件队列
await时节点会插入condition queue（独占访问）
收到signal通知时，节点会被转移到主队列
通过一个字段的特殊值(nextWaiter)来标志节点在哪个队列(SHARED，非SHARED)

acquire->addWaiter, release 操作的是主队列，共享访问, nextWaiter == SHARED

await->ConditionObject.addConditionWaiter, doSignal 操作的是条件队列，独占访问， nextWaiter!=SHARED

### 共享锁和独占锁： 
AQS.doReleaseShared释放
AQS.doAcquireSharedInterruptibly
共享锁，有足够的permit许可时，setHeadAndPropagate会传播给队列中SHARED共享的线程

### 如何实现加锁等待acquire：
```java
  tryAcquire先尝试加锁: 
    如果没有其他线程锁定则本线程锁定(state==0 and !hasQueuedPredecessors()) 
    tryAcquire->compareAndSetState,setExclusiveOwnerThread
  加锁失败则acquireQueued加入等待队列尝试加锁: 
    addWaiter把本线程加入到tail(自旋)
    for自旋 { 
      尝试加锁：前驱节点是头节点 && (state==0且setExclusiveOwnerThread独占AQS成功 or 已经独占了AQS了)
      加锁成功：setHead本节点设置为头节点（清空thread, prev)，为下一个线程取的锁恢复了第一个条件(当前线程.prev为头节点)，返回中断标志.
      加锁失败：则判断阻塞还是自旋shouldParkAfterFailedAcquire
    }
```
### 如何实现释放release:
  tryRelease->setExclusiveOwnerThread将独占线程置为null, setState锁定数减少
  unparkSuccessor唤醒后续有效(t.waitStatus <=0)节点
    如果next为无效节点(s == null || s.waitStatus > 0)，则while从后往前找有效节点

### 为啥从后往前找？ 
https://www.zhihu.com/question/50724462
    看插入的地方就明白了，
    新节点pre指向tail，tail指向新节点，这里后继指向前驱的指针是由CAS操作保证线程安全的。
    而cas操作之后t.next=node之前，可能会有其他线程进来。所以出现了问题，从尾部向前遍历是一定能遍历到所有的节点。

### 判断阻塞等待还是自旋等待shouldParkAfterFailedAcquire:
- 前置节点SIGNAL(-1) return true本线程阻塞，等待释放信号通知
- 前置节点CANCELLED 被取消(ws > 0) 则循环取前面的节点作为前置节点 return false继续自旋
- 前置节点CAS设置为SIGNAL 然后return false继续自旋

### 如何实现取消cancel
http://www.cs.rochester.edu/u/scott/synchronization/

### 如何实现Fair和NonFair: 
NonFair: lock时先尝试compareAndSetState()
Fair: lock前线判断队列是否有其他线程hasQueuedPredecessors()
 

###  AbstractQueuedLongSynchronizer
long类型的原子性需要额外的机制保障

#### 超时机制awaitNanos
while循环 + park

###  重入机制
state=0时为自由
acquire时 增加state
release时 减少state




