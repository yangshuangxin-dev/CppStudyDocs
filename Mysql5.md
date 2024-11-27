# Mysql 的执行计划和慢日志

## 执行计划

Mysql的server层会对sql进行词法句法分析，生成语法树，然后指定一系列的执行计划，优化器选择最佳的执行计划后，调用存储引擎执行数据库操作。

`EXPLAIN` 命令用来查看 SQL 语句的具体执行过程。其原理是模拟优化器执行 SQL 查询语句，从而知道 MySQL 是如何处理 SQL 语句的。

在需要执行sql的前面加上`EXPLAIN`，就可以获取到该sql的执行计划表。其表的Column含义如下：

| Column        | Meaning                                                      |
| ------------- | ------------------------------------------------------------ |
| id            | The SELECT identifier （查询id）                             |
| select_type   | The SELECT type （查询类型）                                 |
| table         | The table for the output row （输出结果集的表）              |
| partitions    | The matching partitions （匹配的分区）                       |
| type          | The join type （表的连接类型）                               |
| possible_keys | The possible indexes to choose（可能使用的索引）             |
| key           | The index actually chosen （实际使用的索引）                 |
| key_len       | The length of the chosen key （索引字段的长度）              |
| ref           | The columns compared to the index （列与索引的比较）         |
| rows          | Estimate of rows to be examined （预估扫描行数）             |
| filtered      | Percentage of rows filtered by table condition （按表条件过滤的行百分比） |
| extra         | Additional information （额外信息，如是否使用索引覆盖）      |

具体的含义如下

### id

select 查询的序列号，包含一组数字，表示查询中执行select 子句或者操作表的顺序，id 号分为三种情况：

1. id 相同，那么执行顺序从上到下。
2. id 不同，id 越大越先执行。
3. id 有相同的也有不同的，id 相同的按 1 执行，id 不同的按 2 执行。

### select_type

主要用来分辨查询的类型，是普通查询还是联合查询还是子查询。

| select_type Value    | Meaning                                                      |
| -------------------- | ------------------------------------------------------------ |
| SIMPLE               | Simple SELECT (not using UNION or subqueries) （简单查询-没有联合查询和子查询） |
| PRIMARY              | Outermost SELECT (最外层select)                              |
| UNION                | Second or later SELECT statement in a UNION （若第二个select出现在union之后，则被标记为union） |
| DEPENDENT UNION      | Second or later SELECT statement in a UNION, dependent on outer query （union或union all联合而成的结果会受外部表影响） |
| UNION RESULT         | Result of a UNION. （从union表获取结果的select）             |
| SUBQUERY             | First SELECT in subquery （在select或者where列表中包含子查询） |
| DEPENDENT SUBQUERY   | First SELECT in subquery, dependent on outer query（subquery的子查询要受到外部表查询的影响） |
| DERIVED              | Derived table（ from子句中出现的子查询，也叫做派生表）       |
| UNCACHEABLE SUBQUERY | A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query（表示使用子查询的结果不能被缓存） |
| UNCACHEABLE UNION    | The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY)（表示union的查询结果不能被缓存：sql语句未验证） |

### table

对应行正在访问哪一个表，表名或者别名，可能是临时表或者union 合并结果集。

1. 具体表名或者表的别名，从具体的物理表中获取数据。
2. 表明为 derivedN 的形式，表示 id 为 N 的查询产生的衍生表。
3. 当有 union result 的时候，表名是 union n1,n2 等的形式，n1,n2 表示参与 **union** 的 id。

### type

type 显示访问类型；采用怎么样的方式来访问数据。效率从好到坏依次为：

system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

- ALL ： 全表扫描。如果数据量大则需要进行优化。
- index：全索引扫描这个比 ALL 的效率要好，主要有两种情况，一种是当前的查询时覆盖索引，即我们需要的数据在索引中就可以索取；另一种是使用了索引进行排序，这样就避免数据的重排序。
- range：表示利用索引查询的时候限制了范围，在指定范围内进行查询，这样避免了 index 的全索引扫描，适用的操作符：=, <>, >, >=, <, <=, IS NULL, BETWEEN, LIKE, OR, IN 。
- index_subquery：利用索引来关联子查询，不再扫描全表。
- unique_subquery：该连接类型类似与 index_subquery，使用的是唯一索引。
- index_merge：在查询过程中需要多个索引组合使用。
- ref_or_null：对于某个字段即需要关联条件，也需要 null值的情况下，查询优化器会选择这种访问方式。
- ref：使用了非唯一性索引进行数据的查找。
- eq_ref ：使用唯一性索引进行数据查找。
- const：这个表至多有一个匹配行。
- system：表只有一行记录（等于系统表），这是 const 类型的特例。

### possible_keys

查询涉及到字段的索引，则这些索引都会列举出来，但是不一定采纳。

### key

实际使用的索引，如果为 NULL，则没有使用索引。

### key_len

表示索引中使用的字节数。查询中使用的索引长度在不损失精度的情况下长度越短越好。

### ref

显示索引的哪一列被使用了，如果可能的话，是一个常数。

### rows

大致估算出找出所需记录需要读取的行数，反映了sql找了多少条数据，该值越小越好。

### extra

提供额外信息。

- using filesort：使用了文件排序。
- using temporary：建立临时表来保存中间结果，查询完成之后把临时表删除。
- using index：采用覆盖索引，直接从索引中读取数据，而不用访问数据表。如果同时出现 using where 表明索引被用来执行索引键值的查找；如果没有，表明索引被用来读取数据，而不是真的查找。
- using index condition：采用索引下推，减少回表次数。
- using where：使用 where 进行条件过滤。
- using join buffer：使用连接缓存。
- impossible where：where 语句的结果总是 false。

### 优化器选择过程

优化器根据解析树可能会生成多个执行计划，然后选择最优的的执行计划，下面的语句查询优化器信息。

```sql
SHOW VARIABLES LIKE 'optimizer_trace';
-- 启用优化器的追踪
SET optimizer_trace = 'enabled=on';-- 执行一条查询语句
SELECT *
FROM information_schema.optimizer_trace;
-- 用完关闭
SET optimizer_trace = 'enabled=off';
SHOW VARIABLES LIKE 'optimizer_trace';
```

## sql语句优化

如果出现sql执行慢时，就需要考虑sql的性能优化，首先需要找到慢的sql语句，可以使用以下三种方式

1. `show processlist`指令显示了有哪些线程在运行，可以帮助识别出有问题的查询语句。
2. SQL 性能分析利器 show profile
3. 开启慢日志查询。

使用SHOW PROCESSLIST查看连接线程，可以查看此时线上运行的 SQL 语句，如果要查看完整的 SQL 语句：`SHOW FULL PROCESSLIST;` 然后优化该语句。

使用SQL 性能分析利器 show profile能清晰的展示每条SQL的持续时间，开启方式如下：

```sql
# 查看是否开启
SELECT @@profiling;
# 设置开启
SET profiling = 1;
# 查看所有 profiles
show profiles;
# 查看query id 为 10 那条查询
show profile for query 10;
# 查看最后一条查询
show profile;
# 最后关闭
SET profiling = 0;
```

慢日志开启方式如下：

- 命令行开启。

```sql
-- 查询慢日志开关
SHOW GLOBAL VARIABLES LIKE 'slow_query%';
SHOW GLOBAL VARIABLES LIKE 'long_query%';
-- 设置开启慢日志
SET GLOBAL slow_query_log = ON;  -- on 开启 off 关闭
SET GLOBAL long_query_time = 4;   -- 单位秒；默认0s；此时设置为4s
```

- 修改配置文件开启。

```shell
slow_query_log = ON
long_query_time = 4
slow_query_log_file = D:/mysql/mysql57-slow.log
```

查找最近10条慢查询日志命令`mysqldumpslow`

```shell
mysqldumpslow -s t -t 10 -g 'select' D:/mysql/mysql57-slow.log
```

找到需要优化的sql语句后，可以从以下3个方向进行分析优化：

1. 索引优化。在where/group by/ order by的字段使用索引。
2. sql语句优化。in/not in 语句优化成联表查询、减少联合查询。
3. 不存储经常修改的数据。如age年龄字段，需要需要存储生日日期。
