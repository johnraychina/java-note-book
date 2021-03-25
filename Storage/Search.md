
搜索引擎技术

Lucene
ELastic Search



# 数据模型

## 构建索引
word-doc-frequency

分词 tokenizer: document -> token

语言处理 Linguistic Processor: token -> term
- lowercase
- stemming
- lemmatization

索引 Indexer: term -> (term, term frequency) + 多个 [docId, doc term frequency]
- 排序
- 合并


## 处理查询

查询语句：词法分析，语法分析，语言处理 -> 抽象语法树AST

筛选使用：AST做筛选
排序使用：score相关系数

Term Frequency (TF)：即此Term在此文档中出现了多少次。tf 越大说明越重要。
Document Frequency (DF)：即有多少文档包含次Term。df 越大说明越不重要。
IDF = log(N/DF)
weight = TF * IDF 

文档
Document = {term1, term2, …… ,term N}
Document Vector = {weight1, weight2, …… ,weight N}

查询
Query = {term1, term 2, …… , term N}
Query Vector = {weight1, weight2, …… , weight N}

相关系数
score = 查询vector 与 Document Vector的 余弦夹角 = (V_D * V_Q) / (|V_D| * |V_Q|)



