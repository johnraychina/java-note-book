



## 

## mysql多版本管理
https://blog.jcole.us/2014/04/16/the-basics-of-the-innodb-undo-logging-and-history-system/
https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html


mysql使用 roll back segment(undolog ) 记录被修改数据的老版本
一方面用于事务回滚
一方面用于事务隔离

mysql的每条记录有4个隐藏字段：
6-byte DB_TRX_ID 最后改动（包括删除操作）这行数据的事务id
a special bit: 删除标志
7-byte DB_ROLL_PTR 回滚指针，指向一条undolog
6-byte DB_ROW_ID 行号， clustered index聚簇索引会包含它

回滚段中的undolog分为insert和update undologs
insert undolog 插入事务提交后就被废弃
update undolog 更新事务提交后，还不能废弃，若有一致性读事务产生这条记录快照时，要给这些事务提供早期版本，所以要等这些事务全都不在时才能废弃。

所以不能把事务拖得太长（包括哪些只读事务），否则InnoDB无法将对应undologs的数据清空，导致rollback segment回滚段被打满。
Commit your transactions regularly, including those transactions that issue only consistent reads. Otherwise, InnoDB cannot discard data from the update undo logs, and the rollback segment may grow too big, filling up your tablespace.


## 主备切换


