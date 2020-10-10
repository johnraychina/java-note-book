Java 并发工具原理

LockSupport park/unpark 调用native方法实现线程阻塞唤醒
ReentrantLock, ReentrantReadWriteLock 锁
CountdownLatch 飞机等人模型
Semaphore 有限资源循环利用
CyclicBarrier 游戏人等人模型

# ReentrantLock 与 Condition
Condition使用场景：有界缓冲，一个线程写满了就等，一个线程读空了就等，ArrayBlockingQueue
与object monitor不同的是：
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



# 底层：基于AQS(AbstractQueuedSynchronizer)实现

AQS底层：基于队列 + volatile + CAS实现 + LockSupport.park/unpark
lock -> AQS.acquire -> addWaiter加到队列tail -> AQS.acquireQueued { for循环tryAcquire + LockSupport.park }
unlock -> AQS.release -> 取next，若无则从tail遍历取等待的线程做 LockSupport.park

