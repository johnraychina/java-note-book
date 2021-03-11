


# 数据模型
https://hbase.apache.org/book.html#datamodel

参见[Google BigTable](Google-Bigtable笔记.md)
参见[HBase and BigTable](https://dzone.com/articles/understanding-hbase-and-bigtab)

- Namespace: 对标数据库
- Table
- Row
- Column Family
- Column Qualifier
- Cell
- Timestamp

**Table**   
An HBase table consists of multiple rows.

**Row**  
A row in HBase consists of a row key and one or more columns with values associated with them. Rows are sorted alphabetically by the row key as they are stored. For this reason, the design of the row key is very important. The goal is to store data in such a way that related rows are near each other. A common row key pattern is a website domain. If your row keys are domains, you should probably store them in reverse (org.apache.www, org.apache.mail, org.apache.jira). This way, all of the Apache domains are near each other in the table, rather than being spread out based on the first letter of the subdomain.

**Column**  
A column in HBase consists of a column family and a column qualifier, which are delimited by a : (colon) character.

**Column Family** 
Column families physically colocate a set of columns and their values, often for performance reasons. Each column family has a set of storage properties, such as whether its values should be cached in memory, how its data is compressed or its row keys are encoded, and others. Each row in a table has the same column families, though a given row might not store anything in a given column family.

**Column Qualifier**
A column qualifier is added to a column family to provide the index for a given piece of data. Given a column family content, a column qualifier might be content:html, and another might be content:pdf. Though column families are fixed at table creation, column qualifiers are mutable and may differ greatly between rows.

**Cell**  
A cell is a combination of row, column family, and column qualifier, and contains a value and a timestamp, which represents the value’s version.

**Timestamp**  

时间戳是和值一同存储的，它代表数据的版本号。
默认情况下，时间戳表示RegionServer上数据写入时间，但你可以在写入时指定一个不同的时间戳。

它是一个long类型的值，通常是java.util.Date.getTime() 或者 System.currentTimeMillis():
当前时间到 January 1, 1970, 00:00:00 GMT 的毫秒数

HBase中版本号是按降序排列，所以最新的数据先被取到。  

hbase是按column family粒度指定存储多少个版本 
create 建表时指定 或者 用 alter 修改  
最多: VERSIONS，最少: MIN_VERSIONS  

Modify the Maximum Number of Versions for a Column Family
```
hbase> alter ‘t1′, NAME => ‘f1′, VERSIONS =>5
```
Modify the Minimum Number of Versions for a Column Family
```
hbase> alter ‘t1′, NAME => ‘f1′, MIN_VERSIONS => 2
```

hbase-site.xml中可以设置全局：hbase.column.max.version

### 核心API
五大基本操作：Get, Scan, Put, Delete, Increment 

#### Get/Scan
get是基于scan实现的

get不指定版本时，默认返回最大版本的值，你修改设置改变返回的版本：
- Get.setMaxVersion()  
- Get.setTimeRanage()  

```java

public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Get get = new Get(Bytes.toBytes("row1"));
get.setMaxVersions(3);  // will return last 3 versions of row
Result r = table.get(get);
byte[] b = r.getValue(CF, ATTR);  // returns current version of value
List<Cell> cells = r.getColumnCells(CF, ATTR);  // returns all versions of this column

```

#### Put
put总会创建cell的一个新版本，版本号通常是RegionServer的currentTimeMillis。  
你也可以指定一个long值，这样你就可以设置为过去、将来、或者 时间无关的一个值。

```java

public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Put put = new Put( Bytes.toBytes(row));
long explicitTimeInMs = 555;  // just an example
put.add(CF, ATTR, explicitTimeInMs, Bytes.toBytes(data));
table.put(put);

```


#### Delete

三种delete:
- 删除一列的某个版本
- 删除列：删除一个列的所有版本
- 删除column family

删除机制：和BigTable一样，由于HBase的StoreFile的不变性，它是给各column family添加一个墓碑记录来标识的，直到垃圾回收时才会物理删除记录。

### 
Sort Order

Column Metadata: HBase没有schema，所以列可以自由扩充，这个工作交给应用程序自己来做。

Join: Yes and No, HBase本身只是个KV数据库，不支持像RDBMS的join, 但你可以自己来做（MapReduce, Nested Loop / Hash Join).

ACID语义保证：
[参见HBase ACID语义](https://hbase.apache.org/acid-semantics.html)
Atomicity: 单行
Consistency: yes
Isolation:  "read committed"
Durability: yes


## HBase and Schema Design
如果你有时间的话，不妨看看
[Ian Varley的硕士论文](papers/No-Relation-The-Mixed-Blessings-of-Non-Relational-Databases2009.pdf)

[HBase KeyValue存储](https://hbase.apache.org/book.html#keyvalue)

[BigTable Schema设计](https://cloud.google.com/bigtable/docs/schema-design)

Robert Yokota的[HBase 应用架构](https://blogs.apache.org/hbase/entry/hbase-application-archetypes-redux)


### Schema创建


# 最佳实践

由于数据按按row key做行分区，按column family做列分区。

*从性能角度考虑*，设计表trade-off的因素: Data Locality(磁盘IO、网络IO)、Load Balancing.

**设置row key**
很大程度上影响扫描速度

**设置column family**
过大会放大磁盘读取IO：本来只需要读一列，但整个都column family都会读取。
过小会放大网络IO：不得不读取多个服务器的数据，然后做合并。






# 面试问题
