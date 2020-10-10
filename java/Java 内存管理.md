
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
分代回收理论假设：1.大多数对象朝生夕死，2.熬过月多次垃圾回收测试的对象越难以消亡; 3.跨带引用假设。
MiniorGC, MajorGC, FullGC, MixedGC
Remembered Set???

垃圾回收算法：
标记-清除 Mark-Sweep
复制 Copy
标记-整理 Mark-Compact

GC Root类型：Stack使用的：Local variable 本地变量, Static variables静态变量，JNI references
GC Root枚举(OopMap记录)
Safe Point 安全点
Safe Region 安全区域：
Remembered Set记忆集 and Card Table卡表
Write Barrie 写屏障
并发可达性分析
    
    Wilson于1994年在理论上证明了，当且仅当以下两个条件同时满足时，会产生“对象消失”的问题，即原本应该是黑色的对象被误标为白色：
    ·赋值器插入了一条或多条从黑色对象到白色对象的新引用；
    ·赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。
    
    解决并发扫描时的对象消失问题，只需破坏这两个条件的任意一个即可：
        增量更新（Incremental Update）
        原始快照（Snapshot At The Beginning，SATB）。

Hotspot垃圾回收器类型：
young: SerialNew, ParNew, ParaScavenge
old: SerialOld, ParOld, CMS
yong & old: G1 

CMS：初始标记GCRoot（stop-the-world) -> 并发标记 -> 重新标记(stop-the-world) -> 并发清除
G1：基于Region（可预测的停顿时间模型）
Shenandoah: 基于Rregion，跨代引用保存在ConnectionMatrix（对标RememberedSet）, ForwardingPointer转发指针+Read Barrier读屏障
ZenGC：基于Region，Colored Pointer染色指针
