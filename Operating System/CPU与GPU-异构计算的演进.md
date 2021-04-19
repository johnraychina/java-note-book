

CPU缓存

MESI: CPU缓存一致性协议

指令拆分：为了减少空闲等待，将指令拆分为原子的操作，不同处理器单元并行执行各自功能。

RISC指令处理： IF - ID - EX - MEM - WB
IF: Instruction Fetch   
ID: Instruction Decode   
EX: Execution   
MEM: Memory Access   
WB: Write Back   

超标量和流水线：
IF  - ID - EX - MEM - WB   
      IF - ID - EX - MEM - WB   
           IF - ID - EX - MEM - WB   
                IF - ID - EX - MEM - WB     
                    IF - ID - EX - MEM - WB    


乱序执行： **一个线程内**指令之间，如果没有依赖则可并行执行（依赖关系怎么计算？不太可能graph）。

流水线和乱序执行机制，如果遇到条件分支，仍然需要等待分支确定后才能继续执行后续指令，怎么办呢？

分支预测：   
    预测成功则可以节约时钟周期，提高CPU利用率。   
    预测失败则需要丢弃，重新执行正确的分支。


SIMD：
[单指令多数据流-知乎](https://zhuanlan.zhihu.com/p/31271788)   
[SIMD-wiki](https://en.wikipedia.org/wiki/SIMD)   
[RISC-V Vector Instructions vs ARM and x86 SIMD](https://medium.com/swlh/risc-v-vector-instructions-vs-arm-and-x86-simd-8c9b17963a31)


片内布局演化：

南北桥 > HyperTransport, QuickPath Interconnect(QPI) -> Ring Bus 

->  Mesh Interconnect Arch


CPU: 实时交互，很多组件要共享，低延迟


GPU: 大吞吐量，大量的核心并行执行
每个GPU由一组SM(Streaming Multiprocessor)组成，SM又包含：
- 线程调度器（Warp Scheduler），每个线程束（Warp）包含32个并行线程，他们使用不同数据执行相同的指令。
- 访问存储单元（Load/Store Queues）：在核心与内存之间传输数据
- 核心（Core)：数值计算
- 特殊件函数单元（Special Functions Unit）
- 寄存器文件（Register File)：存储和缓存数据
- 共享内存（Shared Memory）
- 一级缓存和通用缓存

GPU水平扩容：为了提高系统的吞吐量，新的 GPU 架构不只拥有了更多的核心数量，它还需要更大的寄存器、内存、缓存以及带宽满足计算和传输的需求。


专用核心：
- 张量核心（Tensor Core）：牺牲一定精度，在每个时钟周期执行一次4*4矩阵运算
- 光线追踪（Ray-Tracing Core）：针对光线追踪算法设计特殊电路

- ASIC
- FPGA 可编程电路





