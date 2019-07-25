# 隔离级别

​	隔离级别是数据库系统的基础之一。隔离性是ACID中的I。隔离级别是调整性能，可靠性，一致性和多次经过更改和查询的事务结果的可重复性之间的一个设置。InnoDB提供了SQL92描述的全部4种隔离级别：

​	READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ（默认）, SERIALIZABLE

​	用户可以为一次单独的会话设置隔离级别，也可以用 SET TRANSACTION语句为全部随后的会话设置。查看更多隔离级别设置语法的细节，见...

​	InnoDB支持为每个隔离级别使用不同的锁策略。你可以使用默认的拥有较高一致性的REPEATABLE READ在重要的数据操作上。在对一致性和结果可重复性并没有很高要求的数据上，你可以使用READ COMMITTED 甚至 READ UNCOMMITTED

### 1.隔离级别



​        SQL92标准制定了隔离级别，SQL语法，事务，标准函数等。其中隔离级别的提出，是为了降低事务的一致性以提高并发，这在很多业务场景中是很有意义的。事务单元之间的happen-before关系，抽象起来只有四种。

![1556193438313](https://sagestone.oss-cn-hangzhou.aliyuncs.com/huyu/1556193438313.png)

​         

​		所以数据库的并发能力，也就是这四种情况下的并发能力。当所有的情况都可以并行的时候，此时数据库的并发能力是最强的，但是此时数据库服务不仅不具备一致性甚至连基本的正确性也无法保证。当所有的情况都是串行的时候，此时数据库的一致性达到最强，但是却不具备并发性。所以，隔离级别的不同等级，就是通过调整上述关系的并行来实现的。这里并行的含义是指两个事务单元对同一行数据操作的时候，第一个事务单元（事务单元就是事务，加上单元两个字强调原子性）仍然在执行过程中，第二个事务单元能不能开始执行。那么读读并行，指的就是第一个事务读的时候，第二个事务也可以对同一行数据进行读操作。

​      先复习一下隔离级别：

- Read Uncommitted（读未提交）

在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为脏读（Dirty Read）。

- Read Committed（读提交）

这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这种隔离级别 也支持所谓的不可重复读（Nonrepeatable Read），因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。

- Repeatable Read（可重复读）

这是MySQL的默认事务隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行。不过理论上，这会导致另一个棘手的问题：幻读 （Phantom Read）。简单的说，幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。InnoDB和

[Falcon]: https://rj03hou.github.io/MySQL-Falcon%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E/

会使用多版本并发控制（MVCC，Multiversion Concurrency Control）或间隙锁解决幻读问题。

- Serializable（串行化）

这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。

​		

​     ![1556248487409](https://sagestone.oss-cn-hangzhou.aliyuncs.com/huyu/1556248487409.png)

​       

### 2. Consistent Nonlocking Reads

一致性读，指的是InnoDB使用多版本技术，呈现给query的是数据库在某个时间点的快照。query看到这个时间点之前所有已提交的事务，看不到之后或者未提交的事务。开启的时机是，事务执行了DQL语句。例外：query看到自己同一个事务中的修改。这个例外会导致：如果你更新了表中的一些rows，恰好其他事务在同时也更新了这张表，那么此时你的query看到的数据状态实际上是表中从来没有存在过的。举个栗子，表中字段 :

| id（主键） | number |
| :--------: | :----: |
|     1      |   1    |
|     2      |   2    |

​	Transaction1： 

​								SELECT * FROM TABLE;

​								UPDATE table SET number=22 WHERE id = 2；

​								SELECT * FROM TABLE;

​	这时候查出来的是  

|  id  | number |
| :--: | :----: |
|  1   |   1    |
|  2   |   22   |

但是如果在Transaction 1执行过程中，Transaction 2执行了

Transaction 2 ：UPDATE table SET number=11 WHERE id = 1；

然后先于Transaction 1提交了。

然后再假设一个线程 不间断的执行SELECT * FROM TABLE; 那么此线程看到的这个表只有以下三种状态：

| （初始状态）id | number |
| :------------: | :----: |
|       1        |   1    |
|       2        |   2    |

| （事务2提交之后）id | number |
| :-----------------: | :----: |
|          1          |   11   |
|          2          |   2    |

| （事务1提交之后）id | number |
| :-----------------: | :----: |
|          1          |   11   |
|          2          |   22   |

由于此线程是“不断执行”，所以它看到的这三种状态也就是外界可以查询出的全部状态，但是可以看出这三种情况没有一种是事务1查出来的，也就是说事务1通过拼凑不同数据的版本，瞎编乱造出了一种状态。



当事务隔离级别是REPEATABLE READ ，所有在同一个事务内的一致性读，读的快照都是在第一次读取时候建立的快照。你可以在出现上述问题之前，赶紧提交事务。



当事务隔离级别是READ COMMITTED ，每一个事务内部的一致性读都会更新、读取数据的最新快照。



一致性读是InnoDB在 READ COMMITTED 和REPEATABLE READ 两种隔离级别下执行SELECT时的默认策略。一致性读不会设置任何锁，其他会话可以在同一时间随意修改表。





### 3.MVCC

​	InnoDB是一个支持多版本存储策略的引擎，它保存了被修改记录的旧版本以支持事务。这些旧版本数据被存放在被称作[rollback segment](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_rollback_segment) 的地方。它利用这些旧版本数据支持[一致性读](#2. Consistent Nonlocking Reads)和回滚。

​	MVCC的实现方式具体来说是这样的：InnoDB总是为每个数据行添3个隐藏的字段。

 `DB_TRX_ID`  6-byte 保存了最新的insert和update操作的事务ID。delete对于InnoDB来说也是个update，只是把DB_TRX_ID其中一位做一下标记而已。

`      DB_ROLL_PTR` 7-byte 是一个指针，指向了rollback segment区的undo log记录（如果数据被更新了，undo log中记录着数据被更新之前的信息）

` DB_ROW_ID` 6-byte一个自增的序列，只有聚簇索引会包含这个id的值。如果没有聚簇索引可以认为这列不存在。



​	`rollback segment`中的undo log分为insert-undo log 和 update-undo log.

​	insert-undo log只有在事物回滚时需要，而且只要事务提交了insert-undo log就可以被丢弃。

​	update-undo log也被一致性读用到了，如果没有任何一致性读需要update-undo log中的信息去构建更早版本的数据行，那么update-undo log就可以被丢弃。

​	从上面两条丢弃规则可以看出，我们需要定期提交事务（哪怕只是读事务），否则undo-log会大量堆积浪费磁盘空间。



​      MVCC机制下，即使执行了delete，数据也不会被立马物理删除。删除操作会产生一个undo log 当这个log被废弃的时候才真正delete。



[MySQL官方文档对MVCC的描述]: https://dev.mysql.com/doc/refman/5.7/en/innodb-multi-versioning.html
[segment关于MySQL MVCC的介绍]: https://segmentfault.com/a/1190000012650596



### 4.锁

#### Shared and Exclusive Locks

InnoDB实现了标准的行级锁，有两种类型：

> ​    shared-locks （s）：允许持有此锁的事务读数据

> ​    exclusive-locks （x）：允许持有此锁的事务更新或删除数据

​		如果事务T1持有数据行r的s锁，事务T2可以立马申请第二个s锁，但是无法申请x锁。

​		如果事务T1持有数据行r的x锁，事务T2现在无论申请x锁还是s锁，都不能立马成功，需要等T1释放x锁。

​		只在Serializable级别下默认给SELECT * FROM table 加读锁（隐式的将其转化为 SELECT * FROM table in share model）,其他隔离级别下，可以通过select...lock in share model来显示加锁。

​		SELECT FOR UPDATE加的是X锁。



#### Intention Locks（意向锁）

​		InnoDB支持多种粒度的锁，允许表锁和行锁共存。例如以下语句：LOCK TABLES...WRITE会持有指定表的x锁。为了在支持多个级别锁时更加实用（意思就是提高性能），InnoDB使用意向锁。意向锁是表级别的锁，指明了事务稍后会在行上加什么样的表锁（x or s）。为了快速理解它请先记住这句话：意向锁的作用是在加表级X或S锁之前，优化锁的冲突检测。意图锁分为两种：

​		共享意图锁（IS）：事务想要在表中的某些行设置共享锁。

​		排他意图锁（IX）：事务想要在表中的某些行设置排他锁。

 [`SELECT ... LOCK IN SHARE MODE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) sets an `IS` lock, and [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) sets an `IX` lock.

​		意图锁的规则：

>  事务在获取行的s锁之前，必须先获取IS或者更强（指的应该就是IX）的锁。

>  事务在获取行的x锁之前，必须先获取IX锁。

​		**IS和IX是不与任何行级锁冲突的表级锁。**表级IS，IX，X，S之间的冲突关系可以概括为以下表格（截图来自MySQL官网）

![1557146330423](https://sagestone.oss-cn-hangzhou.aliyuncs.com/huyu/1557146330423.png)

​		Conflict：当T1持有某种锁的时候，T2能不能立马申请到另外一种锁。

​		意向锁的作用：因为S与X锁之间是冲突的，在事务加表级的X或者S锁之前必须保证表中任何一行数据都没有被其他事务的S或者X锁定。为了避免逐一检查每一行数据，就发明了IS，IX锁。结合意图锁的规则可知，如果表中有任何一行数据被S（X）锁定，表上必然会存在一个IS（IX）锁，这样就直接知道无法获得表级X或S。避免了逐一检查每行数据行的锁。意向锁的作用可以总结为：加表级X或S锁之前，优化锁的冲突检测。

​		相关阅读：  [掘金](https://juejin.im/post/5b85124f5188253010326360 "掘金")  [知乎](https://www.zhihu.com/question/51513268)



#### Record Locks

​		例如SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;这个语句保证其他事务无法`插入、更新或修改`t.c1=10的记录。只锁索引而且必须是主键或唯一索引，否则，就不加Record Locks了，换成加Next-Key Locks。



#### Gap Locks

​		间隙锁，锁的是索引区间。注意被锁定的是区域而不是具体的某个索引，区域中有一个索引、多个索引，没有索引都无所谓。被锁住的区域被禁止插入值。如果查询的条件是等值的并且命中的是唯一索引[^a]，就不需要加Gap Lock（用的是上面的记录锁）。它只工作在RR或者更高的级别下，为了解决幻读问题。

​		间隙锁之间是不会冲突的，虽然确实有gap S-lock和gap X-lock之分，但是它们的功能是相同的。只在REPEAD READ级别下并且设置了开启参数：innodb_locks_unsafe_for_binlog = 0；如果不满足这两个条件，Gap Lock会在重复主键检测和外键约束的时候发挥作用。和其他几个锁一样，Gap锁只能锁在索引之间的Gap上这就意味着每个索引都有一堆自己的Gap。



#### Next-Key Locks

​		记录锁+间隙锁=Next-Key Locks。

​		InnoDB像这样实现行级别锁定：当它扫描表上的索引的时候，设置X-lock或S-lock在需要的索引记录上。也就是说，行级别的锁实际上是锁索引记录。Next-Key Locks除了影响索引本身之外，还影响索引之前的gap。举个例子(id age都是索引列)：

![1561868200612](https://sagestone.oss-cn-hangzhou.aliyuncs.com/huyu/1561868200612.png)

上图表示了现在表上存在的gap，Gap key也就只能锁这几个Gap，如果使用的是Next-Key locks就会再加上索引记录本身。例如这样一条语句：SELECT * FROM table WHERE id >6 FOR UPDATE，此时的加锁情况是在id=8这条记录上加上Next-Key Locks，这意味着此时也阻止了id=5的数据被插入。



#### Insert Intention Locks

​		插入意图锁。是一种Gap Lock，一个事务执行INSERT之前，必须获得目标Gap上的插入意图锁。更直白的表述就是，上文的Gap Lock自己和自己不互斥，那么和谁互斥呢？就是这个插入意图锁。



### 5.隔离级别的实现原理

​	



直接跳到重点...

....



下面描述了MySQL是怎么支持不同的事务级别的。

- ### REPEATABLE READ

  这是InnoDB默认的隔离级别。

  > 一致性读：在同一个事务中总是读取事务执行第一条DQL时建立的快照。这意味着，如果你在同一个事务中使用多个SELECT语句这些SELECT语句彼此之间也是一致的。详见一致性读。

  > 当前读：更新，删除，SELECT FOR UPDATE 或者SELECT LOCK IN SHARE MODE这些操作在[阻塞式读|当前读]中，加锁取决于语句命中了等值唯一索引查询还是范围条件查询。

  ​	查询条件只使用了唯一索引。InnoDB只锁相匹配的索引而不锁间隙。

  ​	对于其他的查询条件，InnoDB锁住扫描到的索引范围，使用 gap locks 或者 next-key locks阻止其他会话的插入操作。

- ### READ COMMITTED

  > 一致性读：每个事务的一致性读，更新，读取最新的快照。一致性读详见...

  > 当前读：InnoDB只锁住索引记录而不锁间隙。这样允许在被锁住的记录附近插入新的记录。在此隔离级别下，间隙锁只会用于外键一致性检查和重复主键检查。因为间隙锁被禁用了，可能出现幻读问题。

  当前读中，此隔离级别下只支持基于行（row-based）的二进制日志，如果你使用了 binlog_format = MIXED，也会自动变成基于行的日志。 （可能这在说某种日志机制吧

  此隔离级别有额外的几个影响：
  	    在删除或更新语句中，InnoDB只持有将要被删除或更新的行的行锁，在MySQL评估了WHERE条件之后，未匹配的行上的Record Lock会被提前释放。这大大降低了死锁的风险，但它依然会发生。

  ​		在更新语句中，如果一个行已经被锁住了，InnoDB进行半一致性读（ “semi-consistent” read）返回最新的版本给MySQL，然后MySQL就可以决定UPDATE语句中的WHERE条件，是否被匹配到了。如果行被匹配到了（被匹配到了会被更新），MySQL再次读取这一行，这次InnoDB要么锁定它，要么等待它被锁定。  （...这是一个优化策略

​	思考下面的例子：

> CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
> INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);COMMIT;

这个表没有索引，所以查询和索引扫描使用的记录锁锁定的是隐藏列[^b]而不是索引列。下面有两个会话：

>  Session A:
>
> START TRANSACTION;
> UPDATE t SET b = 5 WHERE b = 3;	

​	

>Session B:
>
>UPDATE t SET b = 4 WHERE b = 2;

​	在InnoDB执行这两个更新的时候，第一个事务为每一行先申请了X锁，然后决定要不要修改。如果InnoDB不会修改这一行，就释放锁（上文提到的提前释放）。除此之外,InnoDB持有将要修改的行的锁直到整个事务结束。事务过程受到的影响如下：
​	
​	当使用REPEATABLE READ隔离级别的时候，第一个事务持有读取到的每一行的X锁，不会提前释放。第二个事务在申请锁的时候会阻塞，直到的第一个事务结束。
​	
​	如果使用READ COMMITTED，第一个事务申请每一行的X锁，但是会释放掉那些不会被修改的行上的锁。第二个事务，InnoDB使用半一致性读，返回最新提交的数据给MySQL，然后MySQL可以决定哪些行符更新操作的WHERE条件。
​	
​	但是，如果WHERE条件中包含索引列，并且InnoDB决定使用这个索引，在获取并保留记录锁的时候只考虑索引列。在下列例子中，第一个更新获取并保持b=2这些行上的记录锁。第二个事务在相同的行上尝试获取X锁的时候被阻塞，因为它也使用了定义在b列上的索引。

> ​    CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
> ​	INSERT INTO t VALUES (1,2,3),(2,2,4);
> ​	COMMIT;



>  Session A:
>
> START TRANSACTION;
> UPDATE t SET b = 3 WHERE b = 2 AND c = 3;

​	

> Session B:
>
> UPDATE t SET b = 4 WHERE b = 2 AND c = 4;

​	

- READ UNCOMMITTED

  SELECT语句以非锁定模式执行，但是可能读取的是过期的版本。此外使用这个隔离级别，读是没有一致性的。这也被叫做脏读。除此之外，这个隔离级别和READ COMMITTED差不多。

- SERIALIZABLE

  这个级别和REPEATABLE READ相似，但是如果autocommit关闭了，InnoDB悄咪咪的把所有的SELECT转换成了 [`SELECT ... LOCK IN SHARE MODE`](https://dev.mysql.com/doc/refman/5.7/en/select.html) ；如果开启了，the [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) is its own transaction. It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions. (To force a plain [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) to block if other transactions have modified the selected rows, disable [`autocommit`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_autocommit).)



加锁分析：

​	MySQL查看加锁状态的语句：SELECT *FROM performance_schema.data_locks

​	输入命令后会出现以下结果集：

​	![1561599307884](https://sagestone.oss-cn-hangzhou.aliyuncs.com/huyu/1561599307884.png)

​	一下数据的值，都是和存储引擎相关的，只讨论在InnoDB下的值以及含义。



​	LOCK_TYPE: TABLE-表锁；RECORD-行锁；



​	LOCK_MODE：锁类型，对于InnoDB有 `S[,GAP]`, `X[,GAP]`, `IS[,GAP]`, `IX[,GAP]`, `AUTO_INC`,  `UNKNOWN`。`X`值得是Next key locks，`X,REC_NOT_GAP`指的是Record locks，`X,GAP`指的是Gap locks。`S`与之类似。



​	LOCK_STATUS: 锁的状态。GRANTED-持有中，WAITING-等待锁。



​	LOCK_DATA：锁定的数据（如果有的话），只有使用行锁时有值，否则该值为null。如果锁定的是主键，显示的就是主键值，如果是二级索引，显示的是二级索引+主键值。另外，显示“ supremum pseudo-record” 时表示锁定在表示该列最大值的虚拟记录上。



### 	3.加锁分析：

​	假设现在有一条SQL：SELECT * FROM table_A WHERE id = 10 FOR UPDATE;

​	想知道加锁的过程，还需要知道这些信息：①隔离级别②id上有没有索引，如果有索引索引类型③表中是否已经存在id=10的记录。

​	序列化+主键索引+存在这条数据：

​	可重复读+主键+有数据：只锁定id=10的这行数据 (X,REC_NOT_GAP)

​	可重复读+主键|二级索引+无数据：锁定随机一个范围 (X ,Gap)

​	可重复读+二级索引+有数据：所有匹配的行 （X）

​	可重复读+无索引+无数据|有数据：锁所有数据行（X）

​	

​	读已提交+主键|二级索引+有数据：只锁定id=10的数据 (X,REC_NOT_GAP)

​	读已提交+主键|二级索引+无数据：不加锁

​	读已提交+无索引 + 无数据：不加锁

​	读已提交+无索引 + 有数据：锁命中的数据行，但是看上去是锁所有数据行。理由见上述加锁逻辑。

​	

​	看上去很复杂其实加锁时只遵守两个很简单的原则：

​	1.锁一定是加在索引上的。所以如果该列没有索引但是依然需要加锁，就要锁全表。

​	2.是否需要防止插入。只有Gap可以防止插入，Gap的范围是由索引情况决定，如果没有索引就只能禁止所有插入操作。

​	剩余两种隔离级别比较简单就不需要在这里说明了，可以理解一下上面两条规则验证一下自己的猜想。

#### 

​	

[^a]: 并不包括联合唯一索引
[^b]: 隐藏列详见MVCC