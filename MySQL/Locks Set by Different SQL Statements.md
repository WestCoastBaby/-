### Locks Set by Different SQL Statements



​		当前读中，更新或删除操作，通常设置锁在SQL语句执行过程中扫描到的每一个索引记录上。不管语句中的WHERE条件有没有排除这一行。InnoDB没有记住WHERE中的排除条件，只知道哪一行索引被扫描到了。锁通常是Next-Key Locks。但是gap lock可以被禁用，会导致不用gap。

​		如果命中了二级索引并且被设置了X锁，InnoDB 也会在对应的聚簇索引记录上设置X锁。

​		如果查询语句没命中任何索引，MySQL必须扫描整个表来执行语句，表上的每一行都会被锁，从而阻止所有其他用户的insert操作。所以创建好的索引是非常重要的。

​		SELECT...FROM 是一致性读，读取某个快照，不会设置任何锁除非隔离级别是SERIALIZABLE。对于这种情况下会设置S-Next Key，而且要保证语句命中了唯一索引并且查询的是一行数据。

​		 [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) 或 [`SELECT ... LOCK IN SHARE MODE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) 被扫描到的行上都会加锁，没有被限定包含于结果集中的行上的锁会被释放（例如没达到where的条件）。但是在某些情况下，行也许不会立即被解锁，因为结果中的行和行的源数据之间的关系在查询执行的过程中丢失了。例如在UNION时，被扫描到的行在评估是否满足where条件之前，可能被插入到临时表中。在这种情况中，临时表中的行和原表的关系就丢失了，稍后的行上的锁会持有到查询结束。

​		[`SELECT ... LOCK IN SHARE MODE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) 在查询到的符合条件的所有行上设置共享next key，但是只有命中唯一索引，查询一行数据的时候。

​		[`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) 在查询到的符合条件的所有行上设置排他next key，但是只有命中唯一索引，查询一行数据的时候。对于查询涉及到的索引记录，此语句阻塞其他会话使用  [`SELECT ... LOCK IN SHARE MODE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) 或使了任意事务隔离级别读。一致性读忽略所有的锁所以不受它影响。

​		[`UPDATE ... WHERE ...`](https://dev.mysql.com/doc/refman/5.7/en/update.html) 在查询到的符合条件的所有行上设置排他next key，但是只有命中唯一索引，查询一行数据的时候。 当UPDATE 修改

​		

​				