# MySQL

本文聚焦 MySQL（主要以 InnoDB 为例）的**事务隔离**与**性能优化**：如何定位慢查询、如何选择数据类型与索引、如何读懂 `EXPLAIN`。
**结构导读**：事务隔离 → 性能剖析 → 数据类型 → 索引优化 → 查询优化 → 常用命令 → 参考

---

## 事务隔离级别（Isolation Level）

隔离级别用于权衡并发与一致性。常见四级如下（从弱到强）：

1. **读未提交（Read Uncommitted）**

   事务可以读取其他事务**未提交**的数据，可能产生**脏读（Dirty Read）**。

1. **读已提交（Read Committed）**

   只能读取已提交数据，避免脏读；但可能产生**不可重复读（Non-Repeatable Read）**（同一事务内两次读到不同结果，因为别的事务提交了更新）。

1. **可重复读（Repeatable Read）**

   保证同一事务内对同一行的多次读取结果一致，避免不可重复读；在某些查询模式下仍可能出现**幻读（Phantom Read）**（同一事务内多次范围查询，结果集行数变化）。
   MySQL 默认隔离级别通常为 **Repeatable Read**。InnoDB 通过 **MVCC** 与 **Next-Key Lock** 等机制，在很多情况下可以避免幻读（特别是加锁读/范围更新场景），但不要简单理解为“MVCC 彻底解决所有幻读”。

1. **串行化（Serializable）**

   最高隔离级别，事务近似串行执行，一致性最强，但并发性能最差。

---

## 性能剖析

性能通常被定义为完成某项任务所需的时间度量，也可以理解为响应时间。
在性能优化过程中，捕捉应用程序和数据库查询的响应时间是非常重要的，这可以帮助我们识别潜在的性能问题并采取相应的措施来改善系统的性能。

### 应用程序性能剖析

应用剖析需要结合你所使用的编程语言选择合适的分析工具如：

| 编程语言 | 分析工具 |
| --- | --- |
| PHP | xhprof |
| Go | pprof |

### MySQL查询剖析

#### 慢日志

慢查询日志是定位慢 SQL 的常用工具之一。合理配置后，通常开销可控；是否开启与阈值设置需结合业务与磁盘能力评估。

* 查看慢日志是否开启

```sql
show variables like '%query%'
```

* 设置慢日志临时开启和慢查询时间临界点

```sql
set global slow_query_log = 1; # 开启慢日志
set global long_query_time = 2; # 阈值 2s（设为 0 可记录全部查询，谨慎使用）
```

通过查看慢日志我们就能捕捉到相应时间较慢的查询语句, 进而针对该语句进行优化。
不过有时候因为某些原因如权限不足等, 无法在服务器上记录查询日志。

#### 查询状态

* 使用`SHOW processlist`

可以周期性采样 `SHOW FULL PROCESSLIST`，记录语句首次出现与消失的时间，很多时候已足够定位瓶颈。

```bash
mysql -proot -e 'show processlist'
```

![profiles](https://iscod.github.io/images/mysql_processlist.png)

`SHOW PROCESSLIST` 中 `Command/State` 等列能帮助判断语句处于什么阶段，例如：

1. Sleep：表示连接处于空闲状态，没有正在执行的命令。
1. Query：表示连接正在执行一个查询语句。
1. Connect：表示连接正在建立。
1. Execute：表示连接正在执行存储过程或函数。
1. Sorting result: 线程正在对结果集进行排序, 这通常发生在包含`ORDER BY`子句的查询中。
1. Sending data: 线程在多个状态之间传送数据, 或者在生成结果集, 或者在向客户端返回数据。
1. Binlog Dump：表示连接正在从二进制日志中读取事件。
1. Table Lock：表示连接正在等待表级锁释放。
1. Kill Query：表示连接正在被终止或取消执行的查询。
1. Prepare：表示连接正在准备一个预处理语句。
1. Init DB：表示连接正在切换到指定的数据库。
1. Annotate Rows：表示连接正在生成查询结果的注释行。
1. Analyzing and Statistics: 线程正在收集存储引擎的统计信息, 并生成查询的执行计划
1. Copying to tmp table [on disk]:  线程正在执行查询, 并且将其结果集复制到一个临时表中。这种状态一般要么是在做`GROUP BY`操作, 要么是文件排序操作, 或者是`UNION`操作。
如果这个状态后面还有 `on disk` 标记，表示 MySQL 将内存临时表落盘（通常意味着内存不足或结果集过大），需要关注 `tmp_table_size`/`max_heap_table_size` 以及 SQL 写法。

#### 查询响应时间

* 使用`SHOW PROFILES`

`SHOW PROFILES` 在新版本中逐步弱化/移除，且默认关闭；更推荐使用 **Performance Schema**、`EXPLAIN ANALYZE`（8.0+）等方式。这里保留原示例用于理解。
profiles 默认禁用，可通过 `profiling` 开启（仅用于临时排查）。

```sql
set profiling = 1; # 开启 profiles 记录
```

```sql
show variables like "pro%";
```

`profiles` 会在查询提交时记录剖析信息，并为查询分配从 1 开始的编号。

```sql
show profiles;
```

![profiles](https://iscod.github.io/images/mysql_profiles_1.png)

可以看到`profiles`以很高的精度显示了查询的响应时间

---

## 数据类型优化

MySQL 支持的数据类型很多。选择合适的数据类型对性能至关重要。通用原则：

- **更小的通常更好**：

尽量选择能正确表达业务含义的**最小**类型。更小的类型通常更快：占用磁盘/内存/CPU cache 更少，CPU 处理周期更短。

- **简单就好**：

简单的数据类型的操作通常需要更少的CPU周期。例如, 整型比字符串操作代价更低, 因为字符集多校对规则（排序规则）使字符串比整型比较更复杂。

典型例子：用**整数存 IPv4**（而不是字符串），用 **MySQL 内建时间类型**存日期时间（而不是字符串）。

> IPv4 本质是 32 位无符号整数。`INET_ATON()` / `INET_NTOA()` 可在字符串与整数间转换。

- **尽量避免 `NULL`**：

很多表包含允许为 `NULL` 的列，即使业务并不需要。注意：`NULL` 与空字符串/0 不同；允许 `NULL` 会让索引与统计更复杂，也可能增加存储开销。

如果查询包含可能为 `NULL` 的列，对优化器而言往往更复杂：索引、统计信息与比较语义都需要特殊处理。

> 可为`NULL`的列会使用更多的存储空间, 在MySQL里也需要特殊处理。当可为`NULL`的列被索引时, 每个索引记录都需要一个额外的字节, 在MyISAM里甚至还可能导致固定大小的索引变成可变大小的索引。

#### 整数类型

整型的几种类型有`TINYINT`、`SMALLINT`、`MEDIUMINT`、`INT`、`BIGINT`等。

整数类型可选`UNSIGNED`属性, 表示不允许出现负值。有符号和无符号类型使用相同的存储空间, 并且具有相同的性能, 因此可以根据实际情况选择合适的类型。

> `MySQL`可以为整数类型指定宽度, 例如`INT(11)`, `对大多数应用程序这是没有意义的`。它不会限制值的合法范围, 只是规定了MySQL一些交互工具显示字符的个数。对于存储和计算来说, `INT(1)`和`INT(11)`是相同的。

#### 实数类型

实数是带有小数部分的数字。当然它们不只是为了存储小数部分，也可以使用 `DECIMAL` 存储比 `BIGINT` 更大的整数。
实数类型有精确小数的`DECIMAL`类型, 也有用于浮点计算的`FLOAT`, `DOUBLE`类型。

浮点类型和`DECIMAL`类型都可以指定精度, 对于`DECIMAL`列, 可以指定小数点前后所允许的最大位数, 但是会影响列的空间消耗, 因为小数点本身也占了1字节空间。浮点类型在存储相同范围的值时,通常比`DECIMAL`使用更少的空间。`FLOAT`使用4个字节存储, `DECIMAL`使用8个字节存储。

> 因为需要额外的空间和计算开销所以应该尽量只在对小数进行精确计算时才使用`DECIMAL`类型, 比如存储财务数据。
但在数据量较大时可以采用`BIGINT`进行替代。所以依然建议使用`整数类型`存储财务数据。

#### 字符串类型

字符串包含`CHAR`、`VARCHAR`、`TINYTEXT`、`TEXT`、`LONGTEXT`等

`CHAR`、`VARCHAR`是两个最重要的字符串类型。二者的存储方式和存储引擎有关，不过主流的 `InnoDB` 与 `MyISAM` 在概念上类似。下面的叙述以常见引擎为参考；若使用其它引擎，请以其文档为准。

* CHAR

`CHAR`类型是定长的：MySQL总是根据定义的字符串长度分配足够的空间。

`CHAR`适合存储很短的字符串, 或者所有值都接近一个同一个长度。
例如`CHAR`非常适合存储密码的`MD5`值, 因为这是一个定长的值。对于一个经常变更的数据, `CHAR`也比`VARCHAR`更好, 因为定长的`CHAR`不容易产生碎片。

对与非常短的列`CHAR`比`VARCHAR`在存储空间上也更有效率, 例如`char(1)`用来存储单个字符, 只需要一个字符, 但是`varchar(1)`却需要两个字符, 因为它还需要一个字符用来记录长度。

* VARCHAR

`VARCHAR`用于存储可变长字符串。

大多数情况下`VARCHAR`比定长类型更节省空间, 因为它仅使用必要的空间（越短的字符串使用的空间越少）

`VARCHAR`需要使用1或2个额外字节记录字符串的长度, 如果列的最大长度小于或等于255, 则使用1个字节, 否则使用2个字节

`VARCHAR`适合以下几种情况:

1, 字符串列的最大长度比平均长度大的多

2, 列的更新很少(减少碎片)

3, 使用了 `utf8mb4` 等多字节字符集，每个字符可能使用不同字节数存储


#### 日期和时间类型

时间类型主要有`DATETIME`和`TIMESTAMP`, 两者都能很好的工作, 但是在某一些场景, 一个比另外一个会工作的更好。

* DATETIME

这个类型可以保存大范围的值, 从10001年到9999年, 精度为秒。它把日期和时间封装到格式为YYYYMMDDHHMMSS的整数中, 与时区无关。使用8字节的存储空间。

* TIMESTAMP

`TIMESTAMP`类型保存了从 1970 年 1 月 1 日午夜（UTC）以来的秒数，和 Unix 时间戳类似。`TIMESTAMP`常见只使用 4 字节存储空间，因此范围比 `DATETIME` 小很多，通常为 1970～2038 年（与实现/版本有关）。

MySQL提供了`FROM_UNIXTIME()`函数把`Unix`时间戳转换为日期, 并提供了`UNIX_TIMESTAMP()`函数把日期转换为`Unix`时间戳


> 除特殊行为之外, 通常应该尽量使用`TIMESTAMP`, 因为它比`DATETIME`空间效率高。有时候人们会选择将`Unix`时间戳存储为整数, 但是这样并不能带来任何收益。用整数保存时间戳的格式不方便处理, 因此并不推荐这么做。

#### 位数据类型

位类型（如 `BIT`）适合存储布尔/标志位集合；但在查询与展示上往往不如 `TINYINT(1)` 直观。是否使用取决于读写频率与可读性需求。

#### 选择标识符（主键）类型

主键会出现在二级索引叶子节点中（InnoDB），因此主键越大，二级索引也越大。常见建议：

- 优先使用**短且稳定**的主键（如自增整型或有序 UUID 方案），避免随机主键导致页分裂与写放大。
- 若业务需要全局唯一且对写入顺序敏感，可考虑有序 UUID/雪花 ID 等（需结合分库分表与排序需求）。

## 索引优化

索引（MySQL 中也叫做 `key`）是存储引擎用于快速找到记录的一种数据结构。

理解索引如何工作，常类比一本书的「索引/目录」：想找某个主题，先在目录中定位页码，再跳转到正文。

在 MySQL 中，存储引擎会先在索引中定位到记录，再根据索引记录找到对应的数据行（是否需要回表取决于是否覆盖索引）。

索引可以包含一个或多个列。如果索引包含多个列，那么列的顺序十分重要，因为 MySQL 只能高效地使用索引的最左前缀列。

### 索引类型

索引有很多种类型，可以为不同场景提供更好的性能。在 MySQL 中，索引在**存储引擎层面**而不是服务器层实现，因此不同存储引擎的实现方式可能不同，也不是所有引擎都支持所有类型的索引。

#### B-Tree 索引

当人们讨论索引而未特别指明类型时，多半指 B-Tree 索引。它使用 [B-tree](https://iscod.github.io/#/data_struct/tree?id=_2-3%e6%a0%91%e5%92%8c2-3-4%e6%a0%91%e7%ad%89) 数据结构存储数据，大多数 MySQL 引擎都支持。

存储引擎以不同方式实现 B-Tree 索引，性能也可能不同。例如：`MyISAM`使用前缀压缩让索引更小；`InnoDB`的二级索引叶子节点包含主键值，用于回表。

B-Tree索引适用于全键值、键值范围和键前缀查找。其中键前缀查找只适用于最左前缀的查询。

假设有如下数据表: 

```sql
CREATE TABLE `user` (
  `id` int(10) unsigned NOT NULL DEFAULT '0',
  `last_name` varchar(50) NOT NULL DEFAULT '' COMMENT '姓',
  `first_name` varchar(50) NOT NULL DEFAULT '' COMMENT '名称',
  `dob` date NOT NULL,
  PRIMARY KEY (`id`),
  KEY `last_first_index` (`last_name`,`first_name`, `dob`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  ```

- **全值匹配**

对索引的所有列（`last_name`,`first_name`,`dob`）做等值匹配，例如：
```sql
select * from user where last_name = "li" and first_name = "hua" and dob = "1990-01-01";
```

- **匹配最左前缀**

前面提到的索引可用于查询所有姓为 \"li\" 的人，即只使用索引的第一列（`last_name`），例如:
```sql
select * from user where last_name = "li";
```

- **匹配列前缀**

也可以只匹配某一列的值开头部分, 例如可以查询`last_name`列中"l“开头的姓的人:

```sql
select * from user where last_name like "l%";
```
- **匹配范围**

前面提到的索引可以查询姓名在 "li" 和 "ning" 之间的人，这里也只使用索引的第一列(`last_name`)

```sql
select * from user where last_name between "li" and "ning";
```

- **前缀等值 + 后续范围**

前面提到的索引也可用于查询姓全为 "li" 的人, 且名字是字母开头为 "h" 的人。
即第一列（`last_name`）全匹配，第二列（`first_name`）范围匹配

```sql
select * from user where last_name = "li" and first_name like "h%";
```

- **只访问索引的查询（覆盖索引）**

B-Tree 通常可以支持「只访问索引的查询」，即查询只需要访问索引，而无需访问数据行。这属于覆盖索引场景

---

关于 B-Tree 索引的常见限制：

- **不从最左列开始**：无法高效使用索引。例如上述索引无法只按 `first_name` 或只按 `dob` 高效查找。

- **不能跳过中间列**：例如查 `last_name="li"` 且 `dob="1990-01-01"`，若不限制 `first_name`，通常只能用到最左列。

- **范围条件会截断后续列的利用**：若对某列做范围（`<`/`>`/`BETWEEN`/`LIKE 'x%'`），其右侧列一般无法继续用于索引查找（可能仍能用于索引下推/过滤）。
例如：

```sql
select * from user where last_name = "li" and first_name like "h%" and dob = "1990-01-01";
```
该查询通常只能有效利用索引前两列，因为 `LIKE 'h%'` 属于范围条件。若业务允许，可通过改写条件或调整索引顺序来提升利用率（最终以 `EXPLAIN` 为准）。


#### 哈希索引

哈希索引(hash index)基于哈希表实现, 只有精确匹配索引所有列的查询才有效。
对于每一行数据, 存储引擎都会对所有的索引列计算一个哈希码(hash code), 哈希码是一个较小的值, 并且不同键值的行计算出来的哈希码也不一样。
哈希索引将所有的哈希码存储在索引中, 同时在哈希表中保存指向每个数据行的指针。

在`MySQL`中只有`Memory`引擎显式支持哈希索引。值得一提的是`Memory`引擎是支持非唯一哈希索引的, 这在数据库世界里是比较与众不同的。如果多个列的哈希值相同, 索引会以链表的方式存放多个记录指针到同一个哈希条目中。

哈希索引的常见限制：

- 只包含哈希值和行指针，不包含有序字段值，因此一般不能用索引值做排序，也难以做覆盖索引（视实现而定）。

- 哈希索引数据并不是按照索引值顺序存储的, 所以无法用于排序

- 哈希索引也不支持部分索引列匹配查找, 因为哈希索引始终是使用索引列的全部内容来计算哈希值。例如, 在数据列(A,B)上建立哈希索引, 如果查询只有数据列A, 则无法使用索引

- 哈希索引只支持等值比较查询, 包括 =、IN()、<=>。但不支持任何范围查询, 例如 ```where price > 100```

- 等值查询很快，但冲突多时需要遍历冲突链表逐行比较，性能会退化。
当出现哈希冲突的时候, 存储引擎必须遍历链表中所有的行指针, 逐行进行比较, 直到找到所有符合条件的行

- 如果哈希冲突很多的话, 一些索引维护操作的代价也会很高。例如, 如果在某个选择性很低（哈希冲突很多）的列上建立哈希索引, 那么从表中删除一行时, 存储引擎需要遍历对应哈希值的链表中的每一行, 找到并删除对应行的引用, 冲突越多, 代价越大

#### 全文索引

全文索引是一种特殊类型的索引, 它查找的是文本中关键词, 而不是直接比较索引中的值。
全文搜索和其他几个索引的匹配方式完全不一样。它有许多需要注意的细节，比如停用词、词干和复数、布尔搜索等。全文索引更像搜索引擎做的事情，而不是简单的 `WHERE` 条件匹配。

#### 其他索引类别

在`MySQL`还有很多第三方存储引擎使用不同类型的数据结构来存储索引。
例如: `MyISAM`支持空间数据索引(R-TREE), TokuDB使用分形树索引(fractal tree index), ScaleDB使用 Patricia tries

### 索引策略

正确的创建和使用索引是实现高性能查询的基础。

#### 独立的列

如果查询中的列不是独立的, 则`MySQL`就不会使用索引。`独立的列`是指索引列不能是表达式的一部分, 也不能是函数的参数

例如：
```sql
select * from user where id + 1 = 3;
```
凭肉眼可以看出`where`条件与`id = 2`等价, 但是`MySQL`无法自动解析方程式。这完全是用户行为。我们应该简化`where`条件的习惯, 始终将索引列单独放到比较符号的一侧

#### 前缀索引和索引选择性

有时候索引的字符列很长, 这会使索引变的大且慢, 一种策略是创建一个模拟哈希索引。但有时候这样做还不够。

通常我们可以索引列开始的部分字符, 这样可以大大节约索引空间, 从而提高索引效率, 但这样也会降低索引的选择性。

索引的选择性是指: 不重复的索引值（也称为基数）和数据表的记录总数(#T)的比值, 范围从 1/#T 到 1 之间。索引的选择性越高则查询效率越高, 因为选择性高的索引可以让`MySQL`在查找时过滤掉更多的行, 唯一索引的选择性是 1, 这是最好的索引选择性, 性能也是最好的。

对于`BLOB`, `TEXT`或者很长的`varchar`类型的列, 必须使用前缀索引, 因为MySQL不允许索引这些列的完整长度。
而诀窍在于要选择足够长的前缀以保证较高的选择性。

```sql
SELECT COUNT(*) cnt, LEFT(address, 3) FROM address WHERE 1 GROUP BY address ORDER BY cnt LIMIT 10;
```

通过测试不同的前缀长度, 找到最接近完整列的选择性

```sql
ALTER TABLE address ADD KEY (address(3));
```

前缀索引是一种能使索引更小、更快的有效方法, 但另一方面也有缺点：MySQL无法使用前缀索引做`ORDER BY`和`GROUP BY`, 也无法使用前缀索引做覆盖扫描。

> 有时候后缀索引(suffix index)也很有用途(例如找到某个域名下的所有邮件地址)。MySQL原生并不支持后缀索引, 但是可以把字符串反转后进行存储, 并基于此建立前缀索引。可以通过触发器结合`REVERSE()`来维护这种索引。

#### 多列索引

很多人对多列索引的理解不够, 一种常见的错误是, 为每个列创建独立的索引, 或者按照错误的顺序创建多列索引。

有时会有人建议「把 WHERE 条件里的列都各建一个索引」，这往往不是最优解：最多只能得到“勉强可用”的索引组合（并且会增加写入维护成本）。更好的方式通常是根据查询模式设计**合适的联合索引**。例如：

```sql
select * from user where last_name = "lihua" or first_name = "hua";
```

#### 选择合适的索引顺序

- 不考虑排序和分组

在一个多列`B-Tree`索引中, 索引列的顺序意味着索引首先按照最左列进行排序, 其次是第二列, 等等。

当不需要考虑排序和分组时，将选择性最高的列放在最左侧通常是很好的，这时候索引的作用只是用于优化 WHERE 条件的查找。

对于如下查询为例:

```sql
select * from payment where staff_id = 2 and customer_id = 584;
```

是应该创建 (staff_id, customer_id) 索引还是应该颠倒一下信息？可以通过查询来确定这个表中值的分布情况，并确定那个列的选择性更高

```sql
select count(distinct staff_id)/count(*) as staff, count(distinct customer_id)/count(*) as customer from payment;
+--------+----------+
| staff  | customer |
+--------+----------+
| 0.0885 |  0.9358  |
+--------+----------+
1 row in set (0.00 sec)

```

`customer_id`的选择性更高, 所以答案是将其最为索引列的第一列：
```sql
ALTER TABLE `payment` ADD INDEX customer_staff(customer_id, staff_id);
```

> 经验法则: 将选择性最高的列放在前面通常是很好的


- 如果有分组和范围查询时？

试想一下如果对于下列查询：

```sql
select * from payment where staff_id = 2 and customer_id > 584 and customer_id < 600;
```

对比创建`(customer_id, staff_id)`和`(staff_id,customer_id)`两种索引顺序时的查询信息：

```sql
explain select * from payment force index(customer_staff) where staff_id = 2 and customer_id > 584 and customer_id < 600\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: payment
   partitions: NULL
         type: range
possible_keys: customer_staff
          key: customer_staff
      key_len: 6
          ref: NULL
         rows: 54
     filtered: 2.50
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
```

```sql
explain select * from payment force index(staff_customer) where staff_id = 2 and customer_id > 584 and customer_id < 600\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: payment
   partitions: NULL
         type: range
possible_keys: staff_customer
          key: staff_customer
      key_len: 11
          ref: NULL
         rows: 2
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.01 sec)
```

虽然`customer_id`列拥有更高的选择性，但是使用`customer_staff`索引时，扫描的行数(rows)很多。
这是由于索引范围扫描带来的差异。可参考[扫描索引的范围](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates)。

> 经验法则: 首先索引相等，然后索引范围。

#### 覆盖索引

如果索引叶子节点已经包含查询所需的列，就不必回表读取数据行，这类索引称为**覆盖索引（covering index）**。
常见收益：减少随机 I/O、减少回表次数；常见代价：索引更大、写入更慢（需要维护更多索引列）。



## 查询优化

查询性能低下最基本的原因是访问的数据太多, 某些查询可能不可避免地需要筛选大量数据, 但这并不常见。

大部分性能低下的查询都可以通过减少访问数据量的方式进行优化。

对于低效查询，可以从两步入手：

- **应用层**：是否取了超出需要的数据（太多行或太多列）。典型现象：`select *`、不必要的分页深翻、拉全量再过滤。
- **数据库层**：是否扫描了过多行/回表过多。典型现象：`type=ALL`、`rows` 很大、`Using filesort`/`Using temporary` 等。

这里有一些经典案例:

- 查询了不需要的记录
- 多表关联时返回了全部列
- 总是取出全部的列
- 重复查询相同的数据

`EXPLAIN` 是查询优化的基础工具之一，它能展示访问路径、可能/实际使用的索引、预估扫描行数等信息（MySQL 8.0+ 还可用 `EXPLAIN ANALYZE` 看实际执行耗时与行数）。

![profiles](https://iscod.github.io/images/mysql_explain_1.png)

`EXPLAIN` 常见输出列（不同版本略有差异）：

| 列名 | 含义 | 说明 |
| --- | --- | --- |
| `id` | 查询标识 | 越大通常越先执行（并非绝对）。 |
| `select_type` | 查询类型 | SIMPLE/PRIMARY/SUBQUERY 等。 |
| `table` | 表名 | 当前访问的表。 |
| `partitions` | 分区 | 命中的分区信息（若启用分区）。 |
| `type` | 访问类型 | 常见从好到差：`const`/`eq_ref`/`ref`/`range`/`index`/`ALL`。 |
| `possible_keys` | 候选索引 | 优化器认为可能可用的索引。 |
| `key` | 实际索引 | 最终选择的索引。 |
| `key_len` | 使用长度 | 实际使用到的索引前缀长度。 |
| `ref` | 关联列 | 索引与哪一列/常量比较。 |
| `rows` | 预估扫描行 | 估计值，用于判断是否“扫太多”。 |
| `filtered` | 过滤比例 | 估计有多少行能通过条件过滤。 |
| `Extra` | 额外信息 | 常见如 `Using where`、`Using index`、`Using filesort`、`Using temporary`。 |

## 常用命令

- help

```sql
HELP CONTENTS;
```

- 查看当前运行的进程

```sql
show processlist;
show full processlist;
```

 - 杀死连接/语句

```sql
KILL 9;
```

- explain

```sql
explain select * from user where id in (10);
```

- 整理文件碎片

```sql
optimize table table_name;
```

* 参考
    * [use-the-index-luke](https://use-the-index-luke.com/)