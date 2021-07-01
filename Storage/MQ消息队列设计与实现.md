
https://tech.meituan.com/2016/07/01/mq-design.html


- RocketMQ
- Kafka
- google Pub/Sub
- tencent CMQ


# 消息系统设计
- 推模型 与 拉模型
- 投递保障
- 批量收发
- 延时消息
- 顺序消息
- 消息过滤
- 死信
- 消息回溯
- 权限和身份认证



At most once
最多一次，消息可能会丢失，但是不会重复。

At lease once
最少一次，消息不会丢失，但可能重复。

Exactly once
精确一次
实现起来比较复杂，延迟会增加(ack, retry, 幂等)，光靠消息中间件无法实现，需要依赖其他组件。

# RocketMQ

## 高可用

## 定时消息和延迟消息

Timer + 暂存Schedule_topic

## 顺序消息
注意：顺序消息 与 定时和延时消息是冲突的，而且不能做可靠异步发送。

全局顺序消息
分区顺序消息



## 事务消息
二次确认 + 消息回查
创建Producer的时候，就必须指定LocalTransactionChecker的实现类，处理二次确认的异常。


## 批量消息
