
# 安装

# 使用

## 基本数据结构和用法
https://redis.io/commands
https://www.runoob.com/redis/redis-tutorial.html

- string
- list
- hash
- set
- zset



## 高级用法
- 分布式锁
- bitmap位图
- HyperLogLog
### 分布式锁
set  biz_lock true ex 5 nx   
del  biz_lock   
### HyperLogLog
pfadd key val1 val2

### Bloom Filter 布隆过滤器


pfcount key
### 发布订阅
subscribe channel
publish channel
unsubscribe channel
### 事务
MULTI 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
EXEC 执行所有事务块内的命令。
DISCARD 取消
WATCH 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
UNWATCH 取消监视
事务可以理解为一个打包的批量执行脚本，但批量指令并非原子化的操作，中间某条指令的失败不会导致前面已做指令的回滚，也不会造成后续的指令不做。


### lua脚本
EVAL script numkeys key [key ...] arg [arg ...]

###  bitmap位图
说明：适用于各种Y/N打标

命令: 
getbit key offset value   
setbit key offset   
bitcount key [start end]

### 地理位置
- geoadd：添加地理位置的坐标。
- geopos：获取地理位置的坐标。
- geodist：计算两个位置之间的距离。
- georadius：根据用户给定的经纬度坐标来获取指定范围内的地理位置集合。
- georadiusbymember 根据储存在位置集合里面的某个地点获取指定范围内的地理位置集合。
- geohash 返回一个或多个位置对象的 geohash 值。




# 原理
[《Redis设计与实现》](Redis设计与实现.xmind)


# 常见面试问题

