
## Spark 3.0 SQL Engine


- Adaptive Query Execution(AQE): change the exectution plan at runtime to automatically set # of reducers and join algorithms: 
    - Broadcast Hash Join
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
```
- Dynamic partitioning pruning
- Query compile speedups
- Optimizer hints
- ANSI SQL language support
- Python Usablility: python type hints for Pandas UDFs 
- Performance: 
    - Apache Arrow-based calls to python user code
    - Vectorized SparkR calls(40x faster)
    - New Pandas function APIs

## 其他Spark项目
- Kolas: pandas api over spark
- Delta Lake: reliable table storage
- Scale-out on spark: scikit-learn, HYPEROPT, Joblib
- GLOW: large-scale genomics
- RAPIDS: GPU-accelerated data science
- Visualization: Looker, Redash...
- OSS Spark: ZenProject 专注python易用性（错误信息，API端口，性能，api设计）

## 
## 数据倾斜解法 Data Skewing

# Lakehouse 介绍
https://www.youtube.com/watch?v=o54YMz8zvCY&list=PLTPXxbhUt-YXZP1lpsa7kVwpygTBOyrcs&index=3

Structured transactional layer