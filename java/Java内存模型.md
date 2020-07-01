
# 硬件效率与一致性：CPU, 内存, 缓存
* 缓存一致性协议：MSI, MESI
* CPU乱序执行 Out-of-Order-Execution
* 虚拟机即时编译时的指令重排 Instruction-Reordering


# Java 内存模型概念

## Why 为什么要定义内存模型？
  在java之前，主流语言（像C和C++等）直接使用物理硬件和操作系统的内存模型，在一个平台编写的代码并发完全正常，放到另外一个平台运行可能就无法正常工作，用户需要为每个平台编写一套代码，成本太高了。
为了屏蔽不同平台硬件和操作系统的访问差异性，Java虚拟机规范定义了一套内存模型，来达到不同平台下一致地访问效果。
PS：计算机问题中，遇到难以解决的问题就加一层抽象，熟悉的配方:)

定这个模型需要考虑严谨性，不能有歧义，又必须足够灵活，以便虚拟机的实现足够自由的利用硬件的各种特性（寄存器、高速缓存、特有指令等）。

## What 内存模型是什么？

* 工作内存 Working memory
* 8个内存交互：工作内存 <--> 主存
* volatile 的特殊规则
* long, double 的特殊规则
* 原子性 Atomicity, 可见性 Visibility, 有序性 Ordering
* 指令重排与内存屏障
* 先行发生原则 Happens-Before

围绕如何处理变量的原子性、可见性、一致性，JVM规范定义了一套内存模型和变量访问规则。


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








