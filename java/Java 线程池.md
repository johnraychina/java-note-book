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
## execute 线程池提交任务执行过程
先判断核心线程数
  不足：则创建核心线程，直接在其中运行；
  满了：则判断队列空间
    空余：加到队列
    满了：判断最大线程数
        空余：创建非核心线程
        满了：执行拒绝策略

这里还需要着重指出创建线程方法addWorker中是否可以创建线程的条件：当线程池的状态达到了SHUTDOWN或者之上的状态时候，只有一种情况还需要继续添加线程，那就是线程池已经SHUTDOWN，但是队列中还有任务在排队，而且不接受新任务，这里还可以继续添加线程的初衷是加快执行等待队列中的任务，尽快让线程池关闭。


## adWorker 核心逻辑
- 2种情况return false添加失败： 关闭状态且queue为空；工作线程数超过了corePoolSize或者maximumPoolSize
- for循环 + cas重试 增加ctl的工作线程数

- new Worker调用ThreadFactory创建工作线程（指定第一个任务）
- 线程添加到workers set
- 用mainLock针对 workers set的操作加锁
- 启动工作线程，调用worker.runWorker


#  第二阶段
## 创建完worker工作线程后，工作线程自己会while循环从队列取任务

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
会根据线程池的设置 和 当前运行状态来判断，来block或者timed wait一个任务

block(queue.take)
timed wait(queue.poll带超时时间)



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


## 思考问题：线程池本身是线程安全的吗？
考虑多个线程，都往一个线程池提交任务，运行时状态是什么样的呢？

# 如何实现非核心延迟死亡？

其实并没有一个标志

核心线程与非核心线程：
- 核心线程会循环取队列，直到被中断。
- 非核心线程会循环取队列，会判断keepAliveTime，也会判断中断


  




