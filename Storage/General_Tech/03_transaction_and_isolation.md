# 事务和隔离级别

| 隔离级别缩写 | 中文名称 | 英文全称 | 特性 |
|--------------|----------|----------|------|
| `READ UNCOMMITTED` | 读未提交 | Read Uncommitted | 可能会出现脏读（Dirty Read），即可以读取其他事务尚未提交的数据。 |
| `READ COMMITTED` | 读已提交 | Read Committed | 避免了脏读，但可能会出现不可重复读（Non-repeatable Read），即同一事务中多次读取同一数据可能得到不同结果。 |
| `REPEATABLE READ` | 可重复读 | Repeatable Read | 避免了脏读和不可重复读，但可能会出现幻读（Phantom Read），即事务执行过程中，新插入的行符合查询条件而导致查询结果集发生变化。这是 InnoDB 的默认隔离级别。 |
| `SERIALIZABLE` | 可串行化 | Serializable | 最高隔离级别，避免了脏读、不可重复读和幻读。事务之间完全串行执行，可能导致较多的超时和锁竞争。 |