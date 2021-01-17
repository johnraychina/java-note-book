# 使用brew安装mongodb
## 1、安装mongo server
brew tap mongodb/brew
brew install mongodb-community@4.4
安装社区版

brew services start mongodb-community@4.4
启动服务

brew services stop mongodb-community@4.4
停止服务

## 2、客户端连接mongo server
- 自带命令行：mongo

- 增强版本命令行：mongosh
Improved syntax highlighting.
Improved command history.
Improved logging.



# mongdb概念
instance: 服务器实例
shard: 分片
replicaSet: 副本集
database/database
table/collection
row/document
column/field

# 使用

## 增删改查 CRUD

insert, update, delete等变更操作，区分多条和单条操作(many, one).

还有复合操作，由于mongodb是基于document的，所以操作单条document天然是原子的 ，
有很多眼花缭乱的复合操作：
比如，insert操作会在collection不存在时自动创collection
upsert, findAndReplace....

### 文本搜索 Text Search
mongdb支持文本搜索，需要用到文本索引(text index) 和 $text 操作符。

每个document只能有一个文索搜索索引(text search index)，但这个索引可以覆盖多个字段（包括字符串数组）。
例如，在MyCollection集合的name和description字段上建立文本搜索索引，
db.MyCollection.createIndex( { name: "text", description: "text" } )

然后你就能使用$text操作符执行文本搜索了,
$text操作符会按空格和分隔符来切分搜索文本，并按OR逻辑搜索集合。
db.MyCollection.find({$text:{$search:"java coffee shop"}})

### 地理位置搜索 Geospatial Queries

## 聚合 Aggregation
- aggregation pipepline
- map-reduce function
- single purpose aggregation methods


## 数据模型 Data Models








## 事务 Transactions

## 索引 Indexes

## 分片 Sharding

## 副本 Replication

## 存储 Storage

## spring data mongodb

# 原理
