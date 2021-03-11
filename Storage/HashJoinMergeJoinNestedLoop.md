https://dba.stackexchange.com/questions/937/difference-between-hash-merge-and-loop-join
http://www.jasongj.com/2015/03/07/Join1/
https://tomyrhymond.wordpress.com/2011/10/01/nested-loop-hash-and-merge-joins/

## Nested Loop:
对于被连接的数据子集较小的情况，Nested Loop是个较好的选择。Nested Loop就是扫描一个表（外表），每读到一条记录，就根据Join字段上的索引去另一张表（内表）里面查找，若Join字段上没有索引查询优化器一般就不会选择 Nested Loop。在Nested Loop中，内表（一般是带索引的大表）被外表（也叫“驱动表”，一般为小表——不紧相对其它表为小表，而且记录数的绝对值也较小，不要求有索引）驱动，外表返回的每一行都要在内表中检索找到与它匹配的行，因此整个查询返回的结果集不能太大（大于1 万不适合）。

Complexity: O(NlogM)
适用于：小表非常小，大表有索引。


## Hash Join:
Hash Join是做大数据集连接时的常用方式，优化器使用两个表中较小（相对较小）的表利用Join Key在内存中建立散列表，然后扫描较大的表并探测散列表，找出与Hash表匹配的行。
这种方式适用于较小的表完全可以放于内存中的情况，这样总成本就是访问两个表的成本之和。但是在表很大的情况下并不能完全放入内存，这时优化器会将它分割成若干不同的分区，不能放入内存的部分就把该分区写入磁盘的临时段，此时要求有较大的临时段从而尽量提高I/O 的性能。它能够很好的工作于没有索引的大表和并行查询的环境中，并提供最好的性能。大多数人都说它是Join的重型升降机。Hash Join只能应用于等值连接(如WHERE A.COL3 = B.COL4)，这是由Hash的特点决定的。

Complexity: O(N+M)

适用于：probe phase and build phase, 仅支持等值连接，对大表友好。


## Merge Join:
通常情况下Hash Join的效果都比排序合并连接要好，然而如果两表已经被排过序，在执行排序合并连接时不需要再排序了，这时Merge Join的性能会优于Hash Join。Merge join的操作通常分三步：
　　1. 对连接的每个表做table access full;
　　2. 对table access full的结果进行排序。
　　3. 进行merge join对排序结果进行合并。
在全表扫描比索引范围扫描再进行表访问更可取的情况下，Merge Join会比Nested Loop性能更佳。当表特别小或特别巨大的时候，实行全表访问可能会比索引范围扫描更有效。Merge Join的性能开销几乎都在前两步。Merge Join可适于于非等值Join（>，<，>=，<=，但是不包含!=，也即<>）


- Complexity: O(Nhc+Mhm+J) or O(N+M) if you ignore resource 
- consumption costs
- Last-resort join type
- Uses a hash table and a dynamic hash match function to match rows
-  Higher cost in terms of memory consumption and disk I/O utilization.


## 小结
Nested loop 小表 + 带索引的大表

Merge Join  适合大表，需要按key排列，只需读取key一次，所以不太耗费CPU。

Hash Join 适合大表，但是吃CPU，更吃内存，IO相对较小。