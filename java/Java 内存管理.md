
内存管理：布局、分配、初始化、回收


# 内存布局

Heap 堆
    -- TLAB: thread local allocation buffer 每个线程独占
    -- Yong
        -- Remembered Set记忆集
        -- Eden
        -- S0, S1
    -- Old
Non-heap
    -- Method Area
        -- Class code 类代码
        -- Runtime constant pool 常量池
        -- Static variables 静态变量
        -- JIT code 即时编译器生成的代码
    -- JVM stack
    -- Native stack
    -- Program counter
    -- Direct Memory直接内存应用（NIO）


# 内存分配
TLAB: thread local allocation buffer

2种方式：
- Bump The Pointer 指针碰撞
- Free List 空闲空间列表

内存是否规整 ==决定==>回收方式： 
- 压缩Compact
- 清理Sweep

# 对象初始化
Object分为3个部分：
    Object Header
        -- 类型信息
        -- MarkWord: 锁和GC状态 + hashCode GC分代年龄
    Instance Data
        相同宽度的字段总是被分配到一起存放，在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前
    Padding
        8字节倍数




# 内存回收
分代回收理论假设：
    1.大多数对象朝生夕死，
    2.熬过月多次垃圾回收测试的对象越难以消亡; 
    3.跨带引用假设：跨代引用相对于同代引用来说仅占极少数。

MiniorGC, MajorGC, FullGC, MixedGC


垃圾回收算法：
- 标记清除 Mark-Sweep
- 复制 Copy
- 标记压缩 Mark-Compact




### GC Root类型
Stack使用的：Local variable 本地变量, Static variables静态变量，JNI references

### 枚举GC Root要StopTheWorld，那如何快速枚举？
1、笨方法：遍历栈里所有的变量，逐一进行类型判断，如果是 Reference 类型，则属于 GC Roots。
2、高效方法：从外部记录下栈里那些 Reference 类型变量的类型信息，存成一个映射表 -- 这就是 OopMap 的由来。
在Safe Point记录OopMap

Safe Point 安全点：业务线程执行到安全点，然后等待
Safe Region 安全区域：业务线程不执行（blocked, waiting）


### 跨代引用问题？扫描新生代时，出现老年代->新生代时，如何避免扫描老年代？
抽象：Remembered Set记忆集 
实现：Card Table卡表
一个卡页的内存中通常包含不止一个对象，只要卡页内有一个（或更多）对象的字段存在着跨代指针，
那就将对应卡表的数组元素的值标识为1，称为这个元素变脏（Dirty），没有则标识为0。

**在垃圾收集发生时，只要筛选出卡表中变脏的元素，就能轻易得出哪些卡页内存块中包含跨代指针，把它们加入GC Roots中一并扫描。**

在HotSpot虚拟机里是通过写屏障技术（相当于切面技术）维护卡表状态的。

### 并发标记

一致性视图下遍历对象图
Wilson于1994年在理论上证明了，当且仅当以下两个条件同时满足时，会产生“对象消失”的问题，即原本应该是黑色的对象被误标为白色：
 - 赋值器插入了一条或多条从黑色对象到白色对象的新引用；
 - 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。
因此，我们要解决并发扫描时的对象消失问题，只需破坏这两个条件的任意一个即可。
由此分别产生了两种解决方案：增量更新（Incremental Update）和原始快照（Snapshot At The Beginning，SATB）。


### Write Barrie 写屏障
并发可达性分析
    
    Wilson于1994年在理论上证明了，当且仅当以下两个条件同时满足时，会产生“对象消失”的问题，即原本应该是黑色的对象被误标为白色：
    ·赋值器插入了一条或多条从黑色对象到白色对象的新引用；
    ·赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。
    
    解决并发扫描时的对象消失问题，只需破坏这两个条件的任意一个即可：
        增量更新（Incremental Update）
        原始快照（Snapshot At The Beginning，SATB）。

# Hotspot垃圾回收器类型：
young: SerialNew, ParNew, ParaScavenge
old: SerialOld, ParOld, CMS
yong & old: G1 


- CMS：初始标记GCRoot（stop-the-world) -> 并发标记 -> 重新标记(stop-the-world) -> 并发清除

- G1：基于Region（可预测的停顿时间模型）

- Shenandoah: 基于Rregion，跨代引用保存在ConnectionMatrix（对标RememberedSet）, ForwardingPointer转发指针+Read Barrier读屏障

- ZenGC：基于Region，Colored Pointer染色指针

# G1垃圾回收
https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

In summary, there are a few key points we can make about the G1 garbage collection on the old generation.

1. Concurrent Marking Phase
    - Liveness information is calculated concurrently while the application is running.
    - This liveness information identifies which regions will be best to reclaim during an evacuation pause.
    - There is no sweeping phase like in CMS.

2. Remark Phase
    - Uses the Snapshot-at-the-Beginning (SATB) algorithm which is much faster then what was used with CMS.
    - Completely empty regions are reclaimed.

3. Copying/Cleanup Phase
    - Young generation and old generation are reclaimed at the same time.
    - Old generation regions are selected based on their liveness.

# ZGC 垃圾回收

