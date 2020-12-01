
# 硬件效率与一致性：CPU, 内存, 缓存
https://zhuanlan.zhihu.com/p/269221065
* 缓存一致性协议： MESI
* CPU乱序执行 Out-of-Order-Execution
* 虚拟机即时编译时的指令重排 Instruction-Reordering

CPU cache： 写直达write-through，写回write-back
缓存一致性问题(Cache Coherence)：MESI{写传播, 总线嗅探实现事务的串形化}

MESI协议保证的是CPU之间的cache副本保持一致

JMM内存模型定义了java线程对内存的访问方式。

# Java内存模型(JMM)

## Why 为什么要定义内存模型？
  在java之前，主流语言（像C和C++等）直接使用物理硬件和操作系统的内存模型，在一个平台编写的代码并发完全正常，放到另外一个平台运行可能就无法正常工作，用户需要为每个平台编写一套代码，成本太高了。
为了屏蔽不同平台硬件和操作系统的访问差异性，Java虚拟机规范定义了一套内存模型，来达到不同平台下一致地访问效果。
PS：计算机问题中，遇到难以解决的问题就加一层抽象，熟悉的配方:)

定这个模型需要考虑严谨性，不能有歧义，又必须足够灵活，以便虚拟机的实现足够自由的利用硬件的各种特性（寄存器、高速缓存、特有指令等）。

## What 内存模型是什么？
维基百科：
java内存模型描述了java编程语言中的多线程是怎样与内存交互的。
java内存模型与代码的单线程执行情况，一起提供了java编程语言的语义。

oracle:
多线程的行为可能会让人感到困惑和违反直觉，尤其是在没有正确同步的时候。这章描述了多线程程序的语义；它包括了一些规则，规定了被多线程更新的共享内存可以读取到哪些值。因为这个定义与不同硬件平台体系的内存模型很相似，所以这些语义一般被称为java程序语言内存模型。

简单来说：围绕多线程如何处理变量的原子性、可见性、顺序，JVM规范定义了一套内存模型和变量访问规则：

* 工作内存(cache, register,stack) <--> 主存
* 原子性 Atomicity, 可见性 Visibility, 有序性 Ordering
* 8个内存交互字节码指令：工作内存 <--> 主存: read/load, store/write, lock/unlock, use/assign
* volatile 的特殊规则
* long, double 的特殊规则
* 指令重排与内存屏障: LoadLoad, StoreStore, LoadStore, StoreLoad
* 先行发生原则 Happens-Before 
as-if-serial语义和happens-before这么做的目的，都是为了在不改变程序执行结果的前提下，尽可能地提高程序执行的并行度。

### Happens-Before 
1）程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
2）监视器锁规则：对一个锁的解锁monitorexit，happens-before于随后对这个锁的加锁monitorenter。
3）volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
4）传递性：如果A happens-before B，且B happens-before C，那么Ahappens-before C。
5）线程start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
6）线程join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。


### volatile：通过加内存屏障保证数据可见性。
参考：《Java并发编程的艺术》
参考：http://gee.cs.oswego.edu/dl/jmm/cookbook.html

写volatile变量之前先读入内存数据，写之后写入内存
❑ 在每个volatile写操作的前面插入一个StoreStore屏障。
❑ 在每个volatile写操作的后面插入一个StoreLoad屏障。
StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见了。
这是因为StoreStore屏障将保障上面所有的普通写在volatile写之前刷新到主内存。

❑ 在每个volatile读操作的后面插入一个LoadLoad屏障。
❑ 在每个volatile读操作的后面插入一个LoadStore屏障。

LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。
LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。

### 锁：
  获取锁时，JMM会把该线程对应的本地内存置为无效。从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。
  释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。

### final：避免final被修改，避免final初始化前被读到
1. 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
  - JMM禁止编译器把final域的写重排序到构造函数之外。
  - 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。

写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。

2. 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。
编译器会在读final域操作的前面插入一个LoadLoad屏障。


https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html

内存模型描述了：给定程序  与 它的执行路径，判断执行路径是否合法。
Java内存模型在每个执行中，根据一定规则，检查执行中的read是否有效，检查被read观察到的write是否有效。
内存模型仅仅描述了程序的各种可能的行为。
只要程序产生的执行结果可以被内存模型预测，并不规定如何实现。
这个宽松的约束给了实现者极大的自由来实现代码转换（java->字节码)，例如对行为重排序、去除无效的同步等。

Programs and Program Order
Synchronization Order
Happens-before Order

# Java 线程

## 线程怎么实现的？
进程     P:  Process 
内核线程  KLT: Kernel Level Thread
轻量级进程LWP: Light Weight process
用户线程  UT:  User Thread

内核线程：P --> LWP --> KLT --> 线程调度器(分配CPU时间片) --> CPU
用户线程：UT --> P --> CPU
混合模式


## 线程如何调度的？
抢占式preemptive threads scheduling
协同式cooperative threads scheduling

## 线程状态如何转换？

New -> Runnable

Runnable <--(synchronized)-->  Blocked

Runnable <--(wait/join/park, notify/notifyAll)-->Waiting

Runnable <--(sleep/wait/join/park)--> Timed Waiting

Runnbale --(run结束)--> Terminated

## 线程Thread

## 线程池 ThreadPoolExecutor

## Spring的 ThreadPoolExecutor








