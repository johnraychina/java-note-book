# cpu架构、多核、多线程分时复用
Amdal's Law: 提供了一个富有洞察力的观察：提升一个部分的性能对整体系统有多大影响。
时间占比为p的部分，性能提升k被后，系统速度提升倍数：S = 1/((1-p) + p/k)

# 线程池有哪些状态，如何流转？

一共5个状态：
RUNNING: 运行状态，接受新任务，持续处理任务队列里的任务.
SHUTDOWN: 不接收新的任务提交，但要执行完队列中的任务。
STOP: 不接收新的任务，不执行队列中的任务，中断正在执行的任务。
DYING: 线程池正在停止运作，终止所有任务，销毁所有工作线程。
TERMINATED: 表示线程池已停止运作，所有工作线程已被销毁，所有任务已被清空或执行完毕.

shutDown: running -> shutdown
shutDownNow: 
    running -> stop
    shutdown -> stop

queue和pool都为空: shutdown -> tidying
pool为空: stop -> tidying

## 关键参数

线程池状态ctl：高3位表示线程状态，低29位（约5亿）表示工作线程数。

corePoolSize核心线程数
maximumPoolSize最大线程数
workQueue阻塞队列，注意：如果队列太大，会导致达不到最大线程数（吞吐量可能达不到预期）
keepAliveTime+unit 存活时间
threadFactory 线程工厂，用来创建线程
handler 拒绝策略

# 第一阶段

## 添加工作线程 或 将任务放入工作队列

基本思路：
- 原则1：优先确保达到核心线程数，不足则创建
- 原则2：队列不满放队列中
- 原则3：实在不行再新增非核心线程
- 原则4：最后再走拒绝策略

Tips:
- execute 内的逻辑和 addWorker内的逻辑配合工作
- 新增线程时会有firstTask，所以第一次是用firstTask执行（参见runWorker逻辑）
- 核心线程与非核心线程，仅仅用来判断上下界，不是工作线程的一个状态（保证worker无状态和逻辑通用）: core=true/false

这里还需要着重指出创建线程方法addWorker中是否可以创建线程的条件：当线程池的状态达到了SHUTDOWN或者之上的状态时候，只有一种情况还需要继续添加线程，那就是线程池已经SHUTDOWN，但是队列中还有任务在排队，而且不接受新任务，这里还可以继续添加线程的初衷是加快执行等待队列中的任务，尽快让线程池关闭。



```java
        //  分3步处理

        //  1. 如果核心线程数不足，尝试创建一个并传递firstTask。
        //  addWorker会再校验一遍runState运行状态和workerCount工作线程数，如果校验不通过则返回false来避免创建过多的线程。
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 2. 如果任务成功加入队列，还要double-check一下线程状态是否isRunning，以防线程池关闭了 或者线程都都挂了：
        //  如果线程池挂了，则回滚入队列操作；
        //  如果线程都挂了，则添加线程；
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //3. 如果无法加入队列，addWorker尝试新增工作线程
        // 如果新增失败（说明任务饱和了或者线程池shutdown了），则执行拒绝策略。
        else if (!addWorker(command, false))
            reject(command);
```


## 核心逻辑：addWorker创建工作线程 
- 2种情况return false添加失败： 关闭状态且queue为空；工作线程数超过了corePoolSize或者maximumPoolSize
- 增加ctl的工作线程数：竞争少，不加锁，for循环 + cas乐观重试 

- new Worker调用ThreadFactory创建工作线程（指定第一个任务）
- 线程添加到workers set
- 用mainLock针对 workers set的操作加锁
- 启动工作线程，调用worker.runWorker


#  第二阶段
## 工作线程执行任务：run->runWorker->while循环getTask 
2个核心方法： runWorker + getTask


### runWorker
javadoc的内容非常详实，直接翻译：
worker主循环，一直循环从队列获取任务执行，并对以下case处理：

1、直接带初始任务执行，不用从队列取。
另外，只要线程池在running状态，我们通过getTask拿到任务。
如果getTask返回null，那么工作线程按线程池状态和配置参数来确定是否退出。
其他由外部业务代码抛出的异常导致的退出，completeAbruptly,  这通常导致processWorkerExit替换这个线程。????

2、在执行任何任务之前，会先加锁，防止执行的时候被中断，只有在线程池stopping时才会设置中断。

3、每个任务执行前会调用beforeExecute，这个方法如果抛异常的话，会导致线程挂掉（completeAbruptly=true退出循环）而不会执行任务。

4、beforeExcute正常执行完后，task.run执行任务，任务抛出的异常会给到afterExecute。
我们对RuntimeException, Error和 Throwable 三者做了区分处理。
因为不能在Runnable中再抛出异常，这里会把异常包到Errors中向外透出（给到线程的UncaughtExceptionHandler).
任何抛出的异常都会导致线程挂掉。

5、在task.run任务执行完成后，会调用afterExecute，这个方法可能会抛出异常，从而导致线程挂掉.
根据java语言规范 JLS Sec 14.20，即使task.run抛出异常，这个异常也会生效。

最终的效果是：afterExecute 和  UncaughtExceptionHandler 有一个对用户代码的问题，有精确的信息。

getTask：
会根据线程池的设置 和 当前运行状态来判断，

block或者timed wait一个任务
或者


block怎么实现：(queue.take)
timed wait怎么实现：(queue.poll带超时时间)



private final ReentrantLock mainLock = new ReentrantLock();
mainLock 锁的用处，为了控制对workers工作线程集合的访问和bookkepping：
1、串行中断，避免中断风暴
2、检查largestPoolSize的统计
3、在检查中断权限 和 执行中断的这段时间内，确保workers set稳定

## 异常处理
业务代码出现异常时
和普通线程一样，继续抛出异常
只是finally会走runWorker->processWorkerExit(completedAbruptly = false)移除工作线程。

对于被future包住的异常，会将异常信息封装在future中。


## 思考问题：线程池本身是线程安全的吗？如何保证？需要权衡什么问题？
考虑多个线程操作通一个线程池，运行时状态是什么样的呢？
线程池有3个状态字段：
1、ctl：线程池状态和工作线程数：通过AtomicInteger 通过CAS乐观锁重试策略保证原子性和一致性
2、workers工作线程集合：通过mainLock加锁保证线程安全
3、workerQueue任务队列：通过blocking queue本身来保证线程安全，内部实现同样适用了ReentrantLock加锁。
以上结论都是基于在写并发程序的时候，尽可能的缩小锁的范围，提高代码的吞吐率为基准的。


# 核心线程与非核心线程区别是什么？ 非核心线程能成为核心线程吗？
中断都会导致工作线程退出线程池。

如何实现线程保活？ 如何实现keepAliveTime退出？
通过workQueue.take等待任务，工作线程进入block状态保活。
通过workQueue.poll等待任务，工作线程进入timed wait状态，超时则退出。
如果设置了allowCoreThreadTimeout，核心线程就和非核心线程一样超时退出。


结论：核心线程和非核心线程的定义只是线程池设计逻辑上的抽象，在源码实现的角度来看是没有加以区分的。
如果从设计逻辑上考虑，非核心线程的创建时机是当前线程池线程总数超过了corePoolSize，且非核心线程超过keepAliveTime后会执行线程退出操作，
而核心线程一般会在线程池中一直保活，所以非核心线程也没有机会成为核心线程；
从源码角度来看，两者的不同“身份”影响的是当前线程池能达到的并发状态。


# 线程数设置多少合理？队列设置多大合理？

基本原则：要分析看业务代码的特性：
1、是CPU计算密集型还是 IO密集型；

线程数 =CPU 核数 * [ 1 +（IO 耗时 / CPU 耗时）]

2、要考虑CPU切换线程上下文是有成本，切换成本有时候比自旋要高，所以才会有自旋锁这种操作:)。


3、业务的要求：吞吐量优先 or 及时响应优先
4、在队列中等待处太久，会不会导致大量超时重试？


## 延时执行
ScheduledThreadPoolExecutor
DelayedWorkQueue + Leader-Follower pattern (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to minimize unnecessary timed waiting. 

## ForkJoinPool
http://gee.cs.oswego.edu/dl/papers/fj.pdf
ForkJoinPool + ForkJoinWorkerThread
• Worker threads process their own deques in LIFO(youngest−first) order, by popping tasks.
• When a worker thread has no local tasks to run, it attemptsto take ("steal") a task from another randomly chosenworker, using a FIFO (oldest first) rule. 

### 分析ForkJoinPool
ForkJoinPool几个重要的参数：

commonParallelism 并行度

commonMaxSpares

// Instance fields
volatile long ctl;                   // main pool control
volatile int runState;               // lockable status
final int config;                    // parallelism, mode
int indexSeed;                       // to generate worker index
volatile WorkQueue[] workQueues;     // main registry
final ForkJoinWorkerThreadFactory factory;
final UncaughtExceptionHandler ueh;  // per-worker UEH
final String workerNamePrefix;       // to create worker name string
volatile AtomicLong stealCounter;    // also used as sync monitor


### 分析ForkJoinThread
与普通的线程不同，ForkJoinThread内部维护了自己的task queue
正常运行走LIFO(youngest−first): 首先从queue的tail取task
工作窃取走FIFO(oldest first): 如果队列空了就随机从其他ForkJoinThread的queue中窃取取head task执行

与普通的线程不同，ForkJoinThread是构造方法直接要传ForkJoinPool
与普通的线程不同，ForkJoinThread是deamon守护线程




## VarHandle
