java.util

## Array 数组
System.arrayCopy

## List 链表
## Stack 栈
用Vector实现，可以用Deque(LinkedList,ArrayDeque)替代
 
## Queue 先进先出队列
主要接口：offer, poll, peek
BlockingQueue 线程安全
Deque 双端访问

- PriorityQueue 优先级队列
基于数组 + 平衡二叉堆实现
Priority queue represented as a balanced binary heap: the two 
children of queue[n] are queue[2*n+1] and queue[2*(n+1)]. 

- DelayedWorkQueue 延迟队列


## Map
HashMap hash分桶+红黑树，多线程resize扩容成环死循环等问题

ConcurrentHashMap 通过CAS + segment减少锁

TreeMap 树形结构实现有序性

LinkedHashMap 记录了添加顺序

## Set

TreeSet




# 排序算法

# 搜索算法

# 

# 图算法
## 最短路径
## 遍历算法

