
# 用户-功能特性
1. Following 一个用户关注其他用户
2. Twitting 用户发推特
3. Timeline 时间线: 看到我发的和我关注人发的推特
    - user
    - home
    - search
4. trends 趋势 

# 数据模型
user: user_id, name
follower: user_id, follower_user_id
twitter: id, user_id, content

# 非功能特性
- users:300M+users
- write: 600 tweets/sec
- read：600,000 tweets/sec

- read heavy
- eventual consitency
- storage

# 系统架构
client app -> LB -> Server -> Redis/DB


1. 发twitter/收twitter

websocket推消息

fanout扩散模型
读扩散：把我关注人发的twitter全部拉取，按时间排序显示，缺点：读取会按关注人数扩大。
写扩散：发twitter时，广播copy发给关注我的人，缺点：按关注我的人数扩大。

基本假设：大部分人发twitter少，读twitter请求多，相差几个数量级，另外，大部分人的follower很少。
优化：大V发的推特优先放缓存+读扩散，除了大V之外（比如关注人数超过1万），其他人发的推特用写扩散；

redis缓存：
follower: user_id -> [u1, u2,..]
twitter: user_id -> [t1, t2,...]


2. Trends趋势
找到尖峰peak：比如5分钟内1万HashTag

窗口：5分钟
统计量：转发量

工具：kafka消息 -> spark window聚合处理 -> redis


3. 搜索：搜人、搜内容
反向全文索引：Inverted full text
word - [doc1, doc2,...]

TF(Term Frequency): 词频，一个文档中某个词出现次数，体现了文档与词的关联程度。

DF(Document Frequency): 包含关键字的文档个数
IDF(Inverted Document Frequency): log(TotalDocNum/DF)
如果一个单词越是不常见，只出现在少数文档中，那么IDF值就越高。
如果一个单词越是普遍存在于大部分的文档中，那么IDF值就越低。

可以说，IDF体现了词的罕见度和区分度，鹤立鸡群就是这个意思。

Score = TF * IDF 
如果搜索的词多次出现在一个文档中，而且比较少出现在其他文档中，那么这个文档和搜索词匹配程度就越高。



