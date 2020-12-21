https://github.com/alibaba/Sentinel/wiki/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8



核心类：

## ProcessorSlotChain 
责任链模式，将不同Slot串在一起，从而将不同的功能（限流、降级、系统保护）组合在一起。

shot chain 可以分为两个部分：统计数据构建(statistics) 和 规则判断(rule checking)。
ProcessorSlot(可扩展的SPI)
- NodeSelectorSlot 收集资源调用路径，树状结构存储，根据调用路径来限流降级；
- ClusterBuilderSlot 资源统计信息及调用信息（RT，QPS, Thread count等），这些信息作为多维度限流、降级的依据；
- Stat
## Context
调用链路上下文，贯穿一次调用链路的所有Entry，维持着entranceNode入口节点，当前节点curNode，调用来源origin等信息，context名称即为调用链路入口名称。
尽量对用户透明，所以通过ThreadLocal传递，所以在异步调用的时候需要手动set。

# Entry
每次资源调用创建一个Entry。包含资源名，curNode, originNode等信息.


## Node
各种统计节点：StatisticsNode(基于滑动窗口统计调用量), DefaultNode, ClusterNode, EntranceNode

几种 Node 的维度（数目）：

ClusterNode 的维度是 resource
DefaultNode 的维度是 resource * context，存在每个 NodeSelectorSlot 的 map 里面
EntranceNode 的维度是 context，存在 ContextUtil 类的 contextNameNodeMap 里面
来源节点（类型为 StatisticNode）的维度是 resource * origin，存在每个 ClusterNode 的 originCountMap 里面

## StatisticSlot

StatisticSlot 是 Sentinel 最为重要的类之一，用于根据规则判断结果，进行相应的统计操作。

entry 的时候：依次执行后面的判断 slot。每个 slot 触发流控的话会抛出异常（BlockException 的子类）。
    若有 BlockException 抛出，则记录 block 数据；
    若无异常抛出则算作可通过（pass），记录 pass 数据。

exit 的时候：若无 error（无论是业务异常还是流控异常），记录 complete（success）以及 RT，线程数-1。

记录数据的维度：
- 线程数+1
- 记录当前 DefaultNode 数据
- 记录对应的 originNode 数据（若存在 origin）
- 累计 IN 统计数据（若流量类型为 IN）。


## 限流规则 与 限流器实现
根据限流规则选取合适的限流器
com.alibaba.csp.sentinel.slots.block.flow.FlowRuleUtil#generateRater
```java
    private static TrafficShapingController generateRater(/*@Valid*/ FlowRule rule) {
        if (rule.getGrade() == RuleConstant.FLOW_GRADE_QPS) {
            switch (rule.getControlBehavior()) {
                case RuleConstant.CONTROL_BEHAVIOR_WARM_UP:
                    return new WarmUpController(rule.getCount(), rule.getWarmUpPeriodSec(),
                        ColdFactorProperty.coldFactor);
                case RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER:
                    return new RateLimiterController(rule.getMaxQueueingTimeMs(), rule.getCount());
                case RuleConstant.CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER:
                    return new WarmUpRateLimiterController(rule.getCount(), rule.getWarmUpPeriodSec(),
                        rule.getMaxQueueingTimeMs(), ColdFactorProperty.coldFactor);
                case RuleConstant.CONTROL_BEHAVIOR_DEFAULT:
                default:
                    // Default mode or unknown mode: default traffic shaping controller (fast-reject).
            }
        }
        return new DefaultController(rule.getCount(), rule.getGrade());
    }
```

## Warm Up怎么实现？参考guava实现
WarmUpController
construct(double count, int warmUpPeriodInSec, int coldFactor)

count是从FlowRule来的，表示Flow control threshold count.
有个codeFactor，代表冷启动斜率，默认3表示 = 正常启动的处理qps / 比冷启动处理的qps
冷启动时间warmUpPeriodInSec，代表冷启动到正常运行时间





## 匀速排队怎么实现？漏桶算法
RateLimiterController


## 根据调用方限流
