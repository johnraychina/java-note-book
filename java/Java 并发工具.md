Java 并发工具原理

LockSupport park/unpark 调用native方法实现线程阻塞唤醒
ReentrantLock, ReentrantReadWriteLock 锁
CountdownLatch 飞机等人模型
Semaphore 有限资源循环利用
CyclicBarrier 游戏人等人模型

# ReentrantLock 与 Condition


# 底层：基于AQS(AbstractQueuedSynchronizer)实现

AQS底层：基于队列 + volatile + CAS实现 + LockSupport.park/unpark
lock -> AQS.acquire -> addWaiter加到队列tail -> AQS.acquireQueued { for循环tryAcquire + LockSupport.park }
unlock -> AQS.release -> 取next，若无则从tail遍历取等待的线程做 LockSupport.park

