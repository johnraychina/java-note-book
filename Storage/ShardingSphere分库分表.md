http://shardingsphere.apache.org/document/current/cn/overview/

# 使用

# 原理

# 关键设计

# 问题

跨库join需要做合并
order by limit
聚合操作（group，sum, count, avg, having等），结果集如何合并？
order by与group by保持一致，避免使用临时表

# 最佳实践
查询尽可能带上分库条件：单库join，单库事务，单库查询

Join时：尽可能不要跨库join，分库建=分库join条件

聚合：聚合排序不冲突（避免用临时表）

