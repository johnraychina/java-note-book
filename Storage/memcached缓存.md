https://tech.meituan.com/2017/03/17/cache-about.html


集群：客户端采用一致性哈希计算node。

一致性哈希：
- 实体节点，比如0~7
- 虚拟节点，比如0~1023，取模映射到实际节点，相当于实际节点打散到这个虚拟节点
- 实体节点如果出现故障，要将对应的虚拟节点remove掉
- hash(key)%1024, 然后TreeMap.ceilingEntry 得到最近可用的虚拟节点，然后映射到实体节点


常用一致性hash算法，每个都值得细细研究：
- murmur
- crc 
- fnv 
- native
- ketama


memcached内部存储： slab + chunk



一天多少秒：24*3600 = 86400， 2^



# redis+本地缓存的亲和性，一致性哈希如何最大化利用本地缓存？