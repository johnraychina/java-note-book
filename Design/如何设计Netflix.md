

# 功能特性
- 上传
- 搜索
- 播放


# 非功能特性
网络延时
计算：编解码、转码、压缩解压
存储容量

# 系统架构
微服务
Zuul 微服务网关
Hystrix 微服务熔断
Eureka: 微服务注册发现
ribbon: 微服务RPC 通信

EVCache: 缓存集群，利用SSD
MySQL: 业务数据
Cassandra: 历史数据
Kafka + 转码器

Chukwa 大数据处理
ElasticSearch: 搜索
Spark: 大数据处理和机器学习，做统计、排序、推荐等

# 数据模型

视频存取、编解码、分发、加速

- Open Connect(Netflix CDN)
- Backend
- Client

数据缓存：一致性哈希

