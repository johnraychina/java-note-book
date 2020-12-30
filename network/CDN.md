https://www.ruanyifeng.com/blog/2016/06/dns.html
先看看DNS域名解析过程：
递归:
Chrome缓存 -> hosts -> Local DNS

迭代：
ask root DNS
ask top DNS
ask ....


https://zhuanlan.zhihu.com/p/28940451

why：速度，全网访问能力

what：4要素：分布式存储、负载均衡、网络请求重定向、内容管理。
内容管理和网络流量管理(Traffic Management)是CDN的核心所在。


关键技术：调度、缓存

## Cache 系统底层技术

多队列网卡CPU亲和性
磁盘IO队列算法
存储模型
内核参数
NUMA架构
CPU L1/L2/L3
协议栈优化

多任务选型
Socket处理
堆与栈
HugePage
GCC/ICC
高性能基础库
SSL/TLS

## cache 选型
Squid
Varnish
Ngnix
OpenResty
ATS
HAProxy

推荐：OpenResty + ATS/Squid，前者解业务，后者负责高性能存储。

