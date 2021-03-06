# MySQL Optimization

正文部分是按照官方原文的翻译，竖线后的内容是自己的想法。



## 数据库级别的优化

- 对于经常更新的业务，要设计多张表，尽可能包含更少的列。对于查询很多的业务。要尽可能包含更少的表，表中有很多列。

  > 读是不会阻塞的但是写会。设计多张小表可以减少阻塞范围。对于查询，减少表的数量在查询数据的时候不但可以减少扫描索引的次数，还可以避免join的笛卡尔积运算。

- 推荐使用InnoDB引擎。

  > InnoDB的优点：细粒度的行级锁及良好的锁策略。数据压缩算法对读写操作都支持，而MyISAM的压缩就只支持只读的表，压缩数据可以减少磁盘占用减少IO次数。InnoDB的崩溃恢复机制很完善。InnoDB支持事物。InnoDB不支持全文索引，而MyISAM支持。

- 内存要够大，能够缓存所有的热数据。对于InnoDB就是Buffer Pool，MyISAM是key cache，MySQL是query cache。

  >    [query cache](https://www.kancloud.cn/thinkphp/mysql-faq/47450)  [官网](https://dev.mysql.com/doc/refman/5.7/en/query-cache.html)

  

## SQL级别的优化

​        数据库应用程序的核心逻辑是通过SQL语句执行的，无论是直接通过解释器发出还是通过API在后台提交。本节中的调整准则有助于加速各种MySQL应用程序。这些准则涵盖了读取和写入数据的SQL操作，一般SQL操作的幕后开销，以及特定方案（如数据库监视）中使用的操作。

​	

### [Optimizing SELECT Statements	](https://dev.mysql.com/doc/refman/8.0/en/select-optimization.html)

​		这里提到了NDB机制，是一种集群的数据处理系统。详情点击这里 [Conditions for NDB pushdown joins](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster-options-variables.html#ndb_join_pushdown-conditions)



- 建立合适的索引。可以使用[`EXPLAIN`](https://dev.mysql.com/doc/refman/5.7/en/explain.html)  分析查询的执行细节，它可以帮助你解决各种复杂的性能问题，写出复杂SQL之后建议首先使用这个命令检查一下执行情况。具体如何使用索引的，后面会详细介绍。
- 包含函数调用的查询，其中函数调用可能只执行一次，可能在每个结果行上都执行一次，甚至在每个涉及到的行的都执行一次。所以要尽可能的隔离查询的每个部分，例如分两次查询，函数替换成常量。
- 对于大表，避免全表扫描。（how?）
- 定期执行 [`ANALYZE TABLE`](https://dev.mysql.com/doc/refman/5.7/en/analyze-table.html)  ，查询优化器会根据解析的结果知道以后对于该表的查询该如何以最优的方案执行，由于表中的数据是会改变的所以要定期执行它。
- 学习调优技术，索引技术和配置参数。这对于每个引擎是不一样的。[“Optimizing InnoDB Queries”](https://dev.mysql.com/doc/refman/5.7/en/optimizing-innodb-queries.html) 。
- 对于single-query，也有一套调优方案：[“Optimizing InnoDB Read-Only Transactions”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-ro-txn.html).
- 在大内存情况下SQL执行的很快，也可以考虑继续优化它以便让你的程序能够在小内存环境下运行的良好。



#### [WHERE Clause Optimization](https://dev.mysql.com/doc/refman/8.0/en/where-optimization.html)

官方建议如果有些SQL为了易读而写的冗余了，没关系MySQL会进行自动优化，包括以下情况：

- 移除不必要的括号，例如：

  ```
   ((a AND b) AND c OR (((a AND b) AND (c AND d))))
  -> (a AND b AND c) OR (a AND b AND c AND d)
  ```

  为啥不移除最左边这组括号？我觉得可以移除了啊！

- 常量展开：

  ```
   (a<b AND b=c) AND a=5
  -> b>5 AND b=c AND a=5
  ```

  

- 移除常量表达式：

  ```sql
   (b>=5 AND b=5) OR (b=6 AND 5=5) OR (b=7 AND 5=6)
  -> b=5 OR b=6
  ```

- Constant expressions used by indexes are evaluated only once. （我不太明白是什么意思直接贴原文好了

- 条件中包含了1=2这样的否定常量表达式，MySQL会直接返回 0 行数据。

- 如果没有适用GROUP BY或者聚合函数，HAVING中的条件会被合并到WHERE中去。

- For each table in a join, a simpler `WHERE` is constructed to get a fast `WHERE` evaluation for the table and also to skip rows as soon as possible.

  > 我的理解是，JOIN的时候会调整WHERE和ON的执行顺序和内容，以提高性能

- 所有的constant tables会被先查出来，满足constant tables的条件（或）：

  1.只有0行或1行数据的表；

  2. A table that is used with a `WHERE` clause on a `PRIMARY KEY` or a `UNIQUE` index, where all index parts are compared to constant expressions and are defined as `NOT NULL`.（我不确定到底要不要涉及普通索引

  并且官网举了个例子，以下的表都会被认为是constant tables：

  ```
  SELECT * FROM t WHERE *primary_key*=1;
  SELECT * FROM t1,t2
  WHERE t1.*primary_key*=1 AND t2.*primary_key*=t1.id;
  ```

  > 常量表，可以理解为在这种表上做查询会非常快

- 找到联合查询的最好方式是尝试所有的情况，所以如果ORDER BY 和GROUP BY的是相同的列，就不用去尝试了。

- 这种情况会创建临时表：如果ORDER BY 和GROUP BY的不是相同的列，或者ORDER BY 和GROUP BY包含的列不是第一个被join的表中的列。

- 如果使用 `SQL_SMALL_RESULT` ，MySQL会使用 内存临时表。SQL_SMALL_RESULT的意思就是告诉MySQL，结果会很小，请直接使用内存临时表，不需要使用索引排序 SQL_SMALL_RESULT必须和GROUP BY、DISTINCT或DISTINCTROW一起使用 ，一般情况下，我们没有必要使用这个选项，让MySQL选择即可。类似的语句还有：https://www.cnblogs.com/wy123/p/6668859.html

- 覆盖索引不需要二次查询数据的物理位置直接返回索引树的值就行，但是要求所有数据的类型都是numeric。

  > 

- 即使使用了索引列作为查询条件，优化器有时候可能会认为全表扫描更优越。有一种情况下，查询结果占了全表数据的30%以上。但是实际情况会非常复杂，优化器会考虑表大小，数据量，IO块大小等。

- Before each row is output, those that do not match the `HAVING` clause are skipped.

  > 这是在说会跳过HAVING不匹配的行吗？这不废话吗？



#### [Range Optimization](https://dev.mysql.com/doc/refman/8.0/en/range-optimization.html)

​	范围访问方法，只使用一个索引查询包含在一个或多个索引值间隔内的表行的子集。它可用于单部分或多部分索引。以下部分描述了优化程序使用范围访问的条件。

> 根据官网的第一段描述，就很不明白这个到底是什么。这其实就是对范围查询时的优化，当where子句中包含了错综复杂的条件时，会对这些条件进行优化，让最终以某个索引为基准进行搜索。我调整一下叙述的顺序，先看一个优化的例子。

下列key1是索引列，nonkey不是索引。

```
SELECT * FROM t1 WHERE
(key1 < 'abc' AND (key1 LIKE 'abcde%' OR key1 LIKE '%b')) OR
(key1 < 'bar' AND nonkey = 4) OR
(key1 < 'uux' AND key1 > 'z');
```

面对这个SQL，优化器会进行以下步骤去优化它：

1.移除`nonkey = 4` 和`key1 LIKE '%b'` 因为他们不能用于索引扫描。移除的方式是替换为true，优化的过程中不会遗漏任何应该出现在结果集中的数据，只会扩大结果集后面再进行过滤。

```
(key1 < 'abc' AND (key1 LIKE 'abcde%' OR TRUE)) OR
(key1 < 'bar' AND TRUE) OR
(key1 < 'uux' AND key1 > 'z')
```

2.将表达式中的ture和false替换掉。例如`key1 LIKE 'abcde%' OR TRUE` = `true`  等，替换之后：

```
(key1 < 'abc') OR (key1 < 'bar')
```

3.合并重叠的区间：

```
 key1 < 'bar'
```

结束。

这个范围条件优化算法，可以运行在任何深度、顺序的OR/AND条件下。这样得到了一个很简洁但是限制条件并不完整的结果集，所以还需要额外的过滤。



这种优化不支持优化空间索引，一种替代方案是使用union手动拆了多个空间索引的条件。

> 明白了范围查询优化是什么之后，下面是详细的定义



##### Range Access Method for Single-Part Indexes

​	single-part索引的范围生效条件的定义如下：

> single-part索引指的是只有一列的索引，对应的multiple-part指的是联合索引

- 对于BTREE和HASH索引，索引和 `常量值` 使用这些操作（ [`=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal), [`<=>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal-to), [`IN()`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_in), [`IS NULL`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_is-null), or [`IS NOT NULL`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_is-not-null) ）

- 对于BTREE索引，索引和常量值使用这些操作（ [`>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than), [`<`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than), [`>=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than-or-equal), [`<=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than-or-equal), [`BETWEEN`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_between),[`!=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-equal),  [`<>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-equal) ）或者使用 [`LIKE`](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#operator_like)  并且相比较的字符串是个常量并且没有以通配符开头

  > 简而言之，能用索引的就能合并，不能用索引的比如上面例子中` key1 LIKE '%b'` 就要替换成true。但是必须注意，否定查询!=也是可以使用索引的，不少网上的资料说否定查询不能使用索引。

- 对于所有的索引类型，多个 [`OR`](https://dev.mysql.com/doc/refman/5.7/en/logical-operators.html#operator_or) or [`AND`](https://dev.mysql.com/doc/refman/5.7/en/logical-operators.html#operator_and) 组成的嵌套条件

常量值的定义如下：

> 感觉常量的定义可以概括为：值是确定的，并且这个值不会因为扫描到的数据行不同而发生变化。

- 查询中的常量

- 来自同一个join中的常量或者系统表

- 不相关的子查询结果

  > 不相关的子查询指的是子查询可以作为一个独立的SQL执行而不依赖外部查询。例如查询某部门中人员详情：
  >
  > ```mysql
  > SELECT *FROM employees WHERE emp_no IN (SELECT emp_no FROM dept_manager WHERE dept_no = 'd002');
  > ```
  >
  > 相关的子查询，子查询的SQL依赖于外部，例如查询某部门中工资大于平均工资的人的工号：
  >
  > ```mysql
  >  SELECT emp_no
  >     FROM salaries e
  >     WHERE e.salary > (SELECT avg(salary) FROM salaries s where s.emp_no = e.emp_no) limit 12;
  > ```
  >
  > 相关非相关子查询 [详情在这里](http://ithare.com/understanding-correlated-and-uncorrelated-sub-queries-in-sql/) 

- 完全由上述类型组成的子表达式

​        某些非常量表达式，也可能被优化成常量表达式。MySQL尝试从每个可能索引的WHERE子句中提取范围条件。在这个过程中，那些不能被用于构建范围条件的条件会被丢弃，产生重叠的条件会被合并，产生空范围的条件会被移除。



##### Range Access Method for Multiple-Part Indexes

​		联合索引的范围优化是对单个索引范围优化的一个扩展。一个在联合索引上的条件约束索引行在一个或多个元组区间内。键元组区间是通过定义索引的顺序在一组键元组上定义的。

> 看完整个介绍之后，它说的优化指的是如何利用联合索引范围查询数据（利用索引优化查询）。而不是对查询条件做优化。第一段说的是联合索引下的数据区间是如何被定义的。例如在单列索引下，key1>5 定义了(5,+∞)这样一个数据区间。那么在联合索引例如 key1>5 and key2='foo'，数据区间是如何被定义的呢？以及更复杂的AND/OR情况下分别是如何定义数区间的？有了数据区间就有了结果集，不同区间取交/并就可以查出最终数据集了。

例如考虑一个联合索引（key1,key2,key3），存在以下数据：

| key1 | key2 | key3 |
| ---- | ---- | ---- |
| NULL | 1    | abc  |
| NULL | 1    | xyz  |
| NULL | 2    | foo  |
| 1    | 1    | abc  |
| 1    | 1    | xyz  |
| 1    | 2    | abc  |
| 2    | 1    | aaa  |

条件key=1定义了如下区间（+inf可理解为极大值）：

```mysql
(1,-inf,-inf) <= (key_part1,key_part2,key_part3) < (1,+inf,+inf)
```

> 这里的等号我觉得没有意义，因为边界值是虚拟的

这个区间覆盖了第4,5,6行数据。可以被用于范围查找。相比之下key3='abc'没有定义一个独立的区间，不能被用于范围查找。

> 理解这句话不妨想一想联合索引时候的树是如何构建的。最左key的权重是最大的，例如(1,10000,1000)在(2,1,1)的左边。在这种机制下权重低的索引如果单独出现会匹配到多个数据区间。回到什么key='abc'的例子匹配到的区间是：(-inf,-inf,abc) <= (key_part1,key_part2,key_part3) < (+inf,+inf,abc) 覆盖了第1,4,6行数据，果然这是很分散的。分散的数据区间就不能用于查询了吗？取并集不行吗？
>
> 不行！原因还在索引的数据结构上。

下面描述指出了更多的细节——联合索引是如何工作在范围查询条件上的。

> 对于HASH索引必须包含和联合索引完全匹配的条件并且必须都是等值查询、

对于BTREE索引。一个区间可能是由AND组合的，每个条件使用=,<=>,IS NULL,<,>,<=,>=,!=,<>,BETWEEN,LIKE 'pattern'(不能以通配符开头的表达式)。可以使用一个区间的必要条件是，此区间可以确定一个匹配条件包含的所有数据行（也可以是两个区间，如果使用!=,<>）



只要操作是=,<=>,IS NULL，优化器就试图使用额外的key部分确定区间。如果操作是,<,>,<=,>=,!=,<>,BETWEEN, LIKE 优化器就不会使用别的key部分了。对于下面这个表达式，优化器使用第一个条件中的=，也会使用第二个条件中的>=，但是不会考虑使用更多的key，所以不会使用第三个条件。

```mysql
key_part1 = 'foo' AND key_part2 >= 10 AND key_part3 > 10
```

这里确定的一个数据区间是：

```sql
('foo',10,-inf) < (key_part1,key_part2,key_part3) < ('foo',+inf,+inf)
```

> 数据区间很可能大于结果集，所以需要额外的筛选

如果查询条件包含OR或AND，数据区间会有并或交操作。之前对于单列索引的条件优化策略，类似的步骤也适用于联合索引。



#### Index Merge Optimization

​		索引合并把多个范围查询的结果集合并成一个，只能作用于来自同一个表的查询。支持union，交叉或union-连接。但是在复杂AND/OR嵌套时无法使用，全文索引无法使用。优化器根据各种成本估计，在不同的索引合并算法和其他访问方法之间进行选择。

​		使用EXPLAIN输出的结果中`type=index_merge`   `key索引列表`   `key_len是索引行数的列表` 。 `Extra` 显示的是使用的合并算法，一共三种：

Using intersect(...)  当一个WHERE条件被转化为使用`AND`的在多个key上的范围条件，并且条件符合：

- n阶联合索引，且所有的索引部分都被覆盖了
- nnoDB中任何覆盖了主键索引的查询



Using union(...)

Using sort_union(...) 

​		

#### IS NULL Optimization

​		`IS NULL` 用来查询为null的值，如果列被定义为`NOT NULL` ，在此列上的 IS NULL 条件会被优化掉。这种优化不会发生在列可能产生NULL的情况下; 例如，如果它来自LEFT JOIN右侧的表。

​		MySQL还可以优化组合`col_name = expr or col_name IS NULL` ，先走索引，在进行一次额外的查询找到为null的行。 当使用此优化时，EXPLAIN显示ref_or_null。

​		优化只能处理一个IS NULL。 在以下查询中，MySQL仅对左边的表达式使用索引，并且无法使用索引b：

```sql
SELECT * FROM t1, t2 WHERE (t1.a=t2.a AND t2.a IS NULL) OR (t1.b=t2.b AND t2.b IS NULL)
```



#### ORDER BY Optimization

​		这里描述了MySQL能否利用索引来加速`ORDER BY` ，如果不能使用索引，就要利用文件排序了。另外ORDER BY是否包含LIMIT，会对返回方式造成不同的影响。

​		MySQL可能会使用索引来加速`ORDER BY`以避免文件排序，即使ORDER BY的列上并没有索引。只要此时使用的是联合索引并且ORDER BY的是联合索引的一部分，并且WHERE中联合索引的其他部分是常量。如果索引没有包含查询语句访问的所有列，只有访问索引比其他访问方法更低耗时才使用索引。 

> ​		排序就是选择一种排序算法（文件排序一般是归并排序）对某数据集进行排序操作。而索引是一组有序的数据，所以利用索引加速排序的原理就是直接按照索引顺序读取数据从而避免排序操作。顺便提一下，如果排序的数据集太大会使用文件排序，体现在`EXPLAIN`时Extra值为Using filesort或Using temporary。
>
> ​		但是这也有一些局限性，比如ORDER BY的是一个二级索引，而要访问的列中包含了除此二级索引之外的列（比如简单的SELECT *），这种情况下我觉得需要进行`二次查询` 才能查出结果列。假设结果集为n，此时即使使用索引，复杂度也达到了$$O(n^2)$$ 。这也就是为什么有时候要比较使用索引和全表扫描的效率。又稍微想了一下，感觉数据库是真复杂：
>
> ​		读取结果集，是按照什么顺序读取的呢？这里主要考虑随机I/O的问题。因为主键就是数据的物理位置，并且页内数据是一块一块紧挨着的。所以按照主键顺序读，可以大大减少随机I/O。那么如果ORDER BY的是二级索引，读取数据的顺序就可能为了避免随机I/O依然按照主键顺序，然后在内存中对数据进行排序。但是考虑到WHERE条件，可能会导致结果集中每一条数据（极端情况）实际上就是分散在不同页中，那么无论怎么读就都是随机I/O了。那么数据库就有理由按照ORDER BY的顺序读取数据，就可以避免上面提到的二次查询的工作。看看后面有没有提到优化器判断开销的依据吧。

​		假设，在（key_part1,key_part_2）有联合索引，下列查询可能使用索引解析ORDER BY。如果不在索引上的列也需要被读取，优化器的实际行为，取决于访问索引是不是比扫描表更有效。

- SELECT * FROM t1 ORDER BY *key_part1*, *key_part2*;

  但是SELECT *可能还包含除key_part1,key_part_2之外的列。这种情况下扫描整个索引，查询那些不包含在索引中的列可能比扫描整个表然后再排序更低效。如果SELECT *只包含这两个索引列，那就会使用索引避免排序。如果t1是InnoDB表，主键是索引的一部分（意思是所有索引中都包含主键值），下面这种语句也可以被优化：SELECT *pk*, *key_part1*, *key_part2* FROM t1 ORDER BY *key_part1*, *key_part2*;

- 这个查询，key_paet1是常量，order by的是key_part2，并且索引（key_part1,key_part_2）的选择度够高让索引扫描比全表扫描更轻松，就会避免排序。

  ```mysql
  SELECT * FROM t1 WHERE key_part1 = constant ORDER BY key_part2;
  ```

  > 想不明白的地方：为什么选择度很高的时候，会出现全表扫描？

- 这个查询，ORDER BY的是key_part2但是查询的每一行数据中都包含了key_part1，所以这个索引也是能用的

  ```mysql
  SELECT * FROM t1
  WHERE key_part1 = constant1 AND key_part2 > constant2
  ORDER BY key_part2;
  ```



某些情况，即使ORDER BY的是索引列，也是确定无法使用索引的

- ORDER BY了不同的索引。

  ```mysql
  SELECT * FROM t1 ORDER BY key1, key2;
  ```

  ORDER BY了联合索引中不连续的部分。

  ```mysql
  SELECT * FROM t1 WHERE key2=constant ORDER BY key1_part1, key1_part3;
  ```

- 升序和降序混用

- 查询条件中使用的索引和ORDER BY中的索引不一样

  > 后面还列举了很多很多种可以和不可以的情况，但是基本原理是一样的：
  >
  > ①能否使用此索引来进行排序
  >
  > 比如本条，where条件中使用的是key1，order by 的是key2，key2显然不能帮助key1排序这就属于无法使用索引排序的情况。
  >
  > ②使用此索引进行排序是否快于使用排序算法进排序
  >



##### 文件排序

​		如果索引不能被用于ORDER BY，MySQL就要使用filesort。filesort是查询语句中一个额外步骤。为了维持内存中的filesort操作，优化器预先分配固定大小的 [`sort_buffer_size`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_sort_buffer_size) 。每一个会话可以改变当前会话的这个数值。

​		如果结果集太大，filesort使用临时表文件。某些类型的查询特别适合于完全在内存中执行文件排序操作。例如带有LIMIT的查询限定了很少的数据集，这在互联网APP中经常出现。

> 官网没介绍文件排序的过程，其实文件排序就是把大量数据分散排序然后将排序后的片段写入文件，然后再把这些文件合并成一个文件。相关介绍网上多。

​		

#### GROUP BY Optimization

​		实现GROUP BY最普通的方法，是扫描整个表，创建临时表，每个组中的数据行都是连续的。然后使用这个临时表执行分组、调用聚合函数的操作。某些情况下，MySQL可以利用索引避免创建临时表。

​		利用索引分组的最重要的先决条件是，所有GROUP BY的列来自同一个索引，并且索引本身是有序的（比如是BTREE而不是 HASH，对于InnoDB总是BTREE）。还取决于，查询中使用了索引中的哪部分，以及聚合函数操作了哪部分。

​		有两个方式通过索引执行GTOUP BY。第一个方法使用所有范围条件执行分组操作，

### 索引优化



​        通常情况下，索引可以加快查询速度。但是MySQL会在决定使用哪个索引上花费时间。在数据集很少的表上索引可有可无，当查询结果包含全表的大部分数据时，索引不如顺序全表扫描速度快。顺序扫描可以使I/O消耗最小化。每个存储引擎对索引数量和索引长度的规定是不一样的，但是所有的引擎的都会大于每张表16个索引，每个索引长度限制不会比256字节更少。



1.MySQL如何使用索引

​    	常见的索引都是使用BTREE，但是有几个例外：空间索引使用R-TREE，内存表支持Hash，InnoDB使用反转列表建立全文索引。

​	一下情况MySQL会使用索引：

- 检索WHERE子句中的条件
- 排除条件。如果有多个索引可供选择，MySQL通常使用找到行数最少的那个（选择度最高）。
- 联合索引的使用要遵守最左前缀。
- join时，使用的字段相同类型和长度时可以使用索引。但是需要注意编码集。
- 在索引列上寻找最小或者最大值时可以使用索引。
- 在group和order 时使用索引。







​		前缀索引：在建索引的时候可以指定只是用字段前面的几个，以减少索引文件的大小，对于BLOB或TEXXT类型，必须指定前缀索引。最长长度是有限制的。如果搜索的条件大于前缀索引的长度，索引也是有用的。只是对结果集需要二次判断。

​		全文索引。只有InnoDB 和 MyISAM支持，可以建立在CHAR VARCHAR TEXT这三种类型上。用于全文扫描。在使用明确的InnoDB单表全文查询时，才会有一些优化。感觉没什么用啊。想了解全文索引看[这个](https://www.cnblogs.com/tommy-huang/p/4483684.html)

​		空间索引也太复杂了吧，涉及到的数据结构（4叉树，R-TREE）和算法（GeoHash ）都没听过。作用大概就是在空间数据（二维及以上），比如二维的坐标系(x,y)利用空间索引可以很方便的查询某个点附近区域的数据。



## 执行计划

​        依据表，列，索引和WHERE子句，MySQL优化器使用很多技术优化查询。在大表上的查询优化可以做到不需要读取所有的行。多表连接可以不需要比较每一个连接行。优化器做的这一系列的操作，被称为“查询执行计划”。

> 注意第一句话，并不是按照SQL的维度统计，如果一个SQL中涉及n个查询，就会出现至少n行结果，如果union还会额外多一行



### Optimizing Queries with EXPLAIN

[`EXPLAIN`](https://dev.mysql.com/doc/refman/5.7/en/explain.html) 语句包含了MySQL是怎么执行语句的：

它可以分析 [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html), [`DELETE`](https://dev.mysql.com/doc/refman/5.7/en/delete.html), [`INSERT`](https://dev.mysql.com/doc/refman/5.7/en/insert.html), [`REPLACE`](https://dev.mysql.com/doc/refman/5.7/en/replace.html),  [`UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/update.html),  table, column.

当[`EXPLAIN`](https://dev.mysql.com/doc/refman/5.7/en/explain.html) 用在一个可以被优化的的语句上，MySQL展示优化器关于执行计划的信息。如果使用`EXPLAIN FORMAT = JSON` 会显示JSON Name中的信息。

> [`DESCRIBE`](https://dev.mysql.com/doc/refman/5.7/en/describe.html) 可简写成 `DESC` 和它功能完全一样，说是D通常被用在表上，E通常被用在语句上。但是这只是个人习惯而已。功能确实一模一样。值得一提的是一般用来分析SQL语句的执行计划但是也可以获取表甚至列的结构信息。

| Column                                                       | JSON Name       | Meaning                                                   |
| ------------------------------------------------------------ | --------------- | --------------------------------------------------------- |
| [`id`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_id) | `select_id`     | 本次查询的标识符                                          |
| [`select_type`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_select_type) | None            | The `SELECT` type                                         |
| [`table`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_table) | `table_name`    | The table for the output row                              |
| [`partitions`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_partitions) | `partitions`    | The matching partitions                                   |
| [`type`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_type) | `access_type`   | The join type                                             |
| [`possible_keys`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_possible_keys) | `possible_keys` | 可能选择的索引                                            |
| [`key`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_key) | `key`           | 实际选择的索引                                            |
| [`key_len`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_key_len) | `key_length`    | The length of the chosen key                              |
| [`ref`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_ref) | `ref`           | The columns compared to the index                         |
| [`rows`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_rows) | `rows`          | 估计应该检查的行数                                        |
| [`filtered`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_filtered) | `filtered`      | 被过滤的行所占比例，最大值是100，也是个估计值，还挺不准的 |
| [`Extra`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_extra) | None            | Additional information                                    |



详细介绍上表中的字段：

- id:从1开始的一个序列号，如果语句中包含多个查询语句，EXPLAIN的结果会出现多行数据。另外有个特殊情况：如果使用`union` 此字段会为null，这种情况下 `table` 等于<unionM,N>表示哪几个id的查询结果被union
- select_type:SELECT的类型，如果是UPDATE语句，值为UPDATE，DELETE同理。下表展示了所有可能的值。

| Column                                                       | JSON Name                  | Meaning                            |
| :----------------------------------------------------------- | -------------------------- | ---------------------------------- |
| SIMPLE                                                       | None                       | 简单查询（没有适用union或子查询）  |
| PRIMARY                                                      | None                       | 最外层的查询                       |
| UNION                                                        | None                       | UNION中的次级查询（和最外层对比）  |
| DEPENDENT UNION                                              | `dependent` (`true`)       | UNION中的次级查询且依赖外部查询    |
| UNION RESULT                                                 | union_result               | UNION的结果                        |
| [`SUBQUERY`](https://dev.mysql.com/doc/refman/5.7/en/optimizer-hints.html#optimizer-hints-subquery) | None                       | 子查询中的第一个查询               |
| DEPENDENT SUBQUERY                                           | `dependent` (`true`)       | 子查询中的第一个查询且依赖外部查询 |
| DERIVED                                                      | None                       | 驱动表                             |
| MATERIALIZED                                                 | materialized_from_subquery | 使具体化的子查询（不知道啥意思）   |
| UNCACHEABLE SUBQUERY                                         | `cacheable` (`false`)      | 不可缓存的子查询（没遇到过）       |
| UNCACHEABLE UNION                                            | `cacheable` (`false`)      | 不可缓存的UNION（没遇到过）        |

`DEPENDENT SUBQUERY` 评估与`UNCACHEABLE SUBQUERY`评估不同。对于`DEPENDENT SUBQUERY`，子查询仅针对来自其外部上下文的变量的每组不同值重新评估一次。对于UNCACHEABLE SUBQUERY，子查询将为外部上下文的每一行重新评估。这里的缓存不同于[QUERY CACHE](https://dev.mysql.com/doc/refman/5.7/en/query-cache-operation.html) ，这里只缓存到SQL执行结束。



- table：查询的表，可以为这些值

  - <union*M*,*N*>：union了哪些查询的结果集，m,n表示上面提到的id。
  - <derived*N*>：该行引用id值为N的行的派生表结果。例如，派生表可能来自FROM子句中的子查询。
  - <subquery*N*>：该行指的是id值为N的行的具体化子查询的结果。

  > 后面两个没写出SQL

- partitions：查询将匹配记录所在的分区。对于非分区表，该值为NULL。

- type：描述了表之间是如何join的，在JSON格式下使用access_type字段。下面按照join类型的最优到最坏分别进行详细说明：

  - [`system`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_system) 表是系统表（只有一条记录）

  - [`const`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_const) 查询发现表中只有一个匹配的行。因为只有一行结果，这个结果可以被查询优化器当成常量。这种情况非常快因为只查询了一次。

    当查询条件包含主键或者唯一索引（如果是联合主键/唯一要比较所有部分）和常量值的比较时就会以const模式连接。

  - [`eq_ref`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_eq_ref) 对于前面表中的每个行，都要从该表中读取一行。这是最常见的链接方式，当连接使用索引并且索引是主键或唯一非空键时会使用此方式。

    eq_ref可以被使用在索引列使用=操作。比较的值可以是一个常数或者一个表达式表示的列，表达式来自之前被查询的表。

  - [`ref`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_ref) 对于前一个表的每个组合，都要读取一遍符合索引的值。此类型被使用，如果join使用最左匹配或key不是主键/唯一，如果key被用来匹配较少的行，这是个不错的模式。

  - [`ref_or_null`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_ref_or_null) 和上一个一样，但是需要为null进行额外的查询。

  - [`index_merge`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_index_merge) 表明使用了索引合并的优化，在这种情况下，输出的key列包含了所使用的索引，key_len包含了索引使用的最长的key的长度。

  - [`unique_subquery`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_unique_subquery) 在某些IN子查询时，替换eq_ref。但是要求索引是唯一的。

  - [`index_subquery`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_index_subquery) 和上面一样，用在索引不唯一时。

  - [`range`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_range) 在给定范围中使用索引查询。key值表示使用了哪个索引，ref固定为null

  - [`index`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_index)  和接下来介绍的ALL是一样的，除了索引已经被扫描过了，有两种情况：

    - 如果索引是覆盖索引，只有索引树会被扫描。这种情况下Extra列显示（Useing index）。
    - 执行全表扫描，使用索引加快order，此时Extra中不会有（Useing index）。

  - [`ALL`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_all) 前一个表执行了全表扫描。通常第一个表没有被标记为const都是不好的。可以通过添加索引避免这种情况。

- possible_keys ：表示了MySQL在这张表中可以选择的key，这一列完全独立于其他列。这里展示的key可能没有被用到。如果此列为null，没有相关的索引。这种情况下你可以通过优化WHRER条件。

- key：

- extra：此字段包含了SQL执行的额外信息，下列解释了可能出现的值（由于太多只挑几个常见的）。

  - Using where 使用了WHERE条件过滤结果集。
  - Using temporary 为了完成查询，MySQL需要建立一个临时表存储结果，典型的情况是GROUP BY和ORDER BY不一样。
  - Using sort_union(...), Using union(...), Using intersect(...) index merge时的三种情况。













