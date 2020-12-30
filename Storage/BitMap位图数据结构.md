

# HyperLogLog

# 高效判重：基于hash+概率的 BloomFilter
https://www.jianshu.com/p/bef2ec1c361f

高效判重，不是100%，有一定精度损失
多hash几次，精度越高，但是性能损失越大，可以调整hash次数。


# bitmap 位图数据结构基本原理

## 介绍
对于数据密集型的应用来说，海量数据的判重和基数统计是两个绕不开的基础问题。

BloomFilter, HyperLogLog 虽然节省空间并且高效，但是付出了一定代价：
- 只能插入元素，不能删除元素。
- 不保证100%准确，总是存在误差。
这两个缺点可以说是所有概率性数据结构(probabilistic data structure)做出的trade-off。

那么有什么想对高效  AND 保证绝对精确的方法呢？ 
答案就在bitmap


## 实现
实现：如果让每个0/1 bit表示一个数字是否存在，反过来将bit的索引位置index作为数值。

优点：那么对于数值是integer整数，原来32个bit表示一个整数，只需要1个bit[index]表示是否存在，压缩率为1:32
由于位图的这个特性，它经常被用在数据库、查询引擎、搜索引擎中，并且位操作之间可以并行，效率更好。

缺点：不管业务中实际的基数有多少，它占用的内存都是恒定不变的。数据越稀疏，空间利用率越低。

如何解决bitmap稀疏存储空间利用率的问题，大佬们提出了多种算法对稀疏位图进行压缩，减少内存占用。
比较有代表性的有：WAH, EWAH, Concise, 以及RoaringBitmap.
前三者都是基于行程长度编码(Run-length encoding, RLE)做压缩的，而RoaringBitmap算是它们的改进版本。


# 高效压缩位图 RoaringBitmap
https://www.jianshu.com/p/818ac4e90daf

应用场景：海量数据的判重和基数统计。
主要思路：将32位无符号整数按高16位分桶，一共2^16=65536个桶，称为container。
存储数据时，按照高16位找到container（找不到就新建一个），再将低16位放入container中。
也就是说一个RoaringBitMap就是很多个container的集合。

## RoaringBitmap container原理

一共三种container：

ArrayContainer
当桶内数据的基数不超过4096时，会采用它来存储，本质上是一个unsigned short类型的有序数组。
数组初始长度为4，随着数据的增多会自动扩容（但最大长度就是4096）。

BitMapContainer
当桶内数据的基数大于4096时，会用它来存储，本质是一个普通位图，用长度固定未1024的unsigned long数组表示。
大小固定：2^16位（8KB），它同样有一个计数器。


RunContainer
初始的RoaringBitmap中没有它，后来才加入的。
它使用可变长度的unsigned short数组存储用行程长度编码（RLE)压缩后的数据。

什么是RLE压缩？
举个例子，连续的整数序列11, 12, 13, 14, 15, 27, 28, 29
会被RLE压缩为两个二元组11, 4, 27, 2，表示11后面紧跟着4个连续递增的值，27后面跟着2个连续递增的值。


由此可见，RunContainer的压缩效果可好可坏，取决于数据分布。
极端情况：所有数据都是连续的，最终只需要4字节（2byte的short表示开始，2byte的short表示值个数）
如果所有数据都不连续，那么不仅不会压缩，反而会加倍，所以她是作为其他2种container的折中方案。


## container转换

先用ArrayContainer，容量超过4096后，自动转为BitmapContainer。

ArrayContainer存储稀疏数据，BitmapContainer存储稠密数据，可以最大限度地避免内存浪费。

RoaringBitmap还可以调用特定API(名为optimize)比较 ArrayContainer/BitmapContainer 与  RunContainer的内存占用。
如果发现压缩的RunContainer较小，就转换为它。


## RoaringBitmap 常见应用场景

Lucene: filter cache缓存文档id
[frame-of-reference-and-roaring-bitmaps](https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps)

Spark: MapStatus组件跟踪存储块释放非空状态

Greenplum: 打标圈选

