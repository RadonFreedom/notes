# MySql 笔记

## 分区表

### 什么是分区表？

> 对用户来说，分区表是一个独立的逻辑表，但是底层由多个物理子表组成。实现分区的代码实际上是对一组底层表的句柄对象（ Handler Object）的封装。
>
> 对分区表的请求，都会通过句柄对象转化成对存储引擎的接口调用。所以分区对于SQL 层来说是一个完全封装底层实现的黑盒子，对应用是透明的，但是从底层的系统来看就很容易发现，每一个分区表都有一个使用＃分隔命名的表文件。

### 为什么使用分区表？

#### 有了索引为什么还用分区表？

> It’s worth restating this: **at very large sizes, B-Tree indexes don’t work.** 
>
> - Unless the index covers the query completely, the server needs to look up the full rows in the table, and that causes random I/O a row at a time over a very large space, which will just kill query response times. 
> - The cost of maintaining the index (disk space, I/O operations) is also very high.



B+树索引将在超大数据量下失去作用，因为：

- 除非索引能正好满足查询条件，查询在得到索引返回的结果范围之后还是得全表扫描，这就造成了大量的随机IO（每次只在很大的一块数据区域内取出一条满足查询条件的数据，因此需要访问很多不同的磁盘块的一小点数据，而不是理想中的从少数磁盘块中取出查询到的大量数据），这将会造成让人无法接受的响应时间。
- 在数据量巨大的情况下，维护索引的开销也是非常高的。因为聚簇索引的根节点存储在内存中，很大的数据量将导致索引有很大的内存占用，更不用说其他索引所占用的磁盘空间和数据有变更时因为要同步修改索引带来的开销。

#### 如何理解分区表？

> The key is to think about **partitioning as a crude form of indexing** that **has very low overhead and gets you in the neighborhood of the data you want.**

可以把分区表当成一种索引的粗糙形式，这种方式下有非常低的开销，并且能快速把你带入到你想要的数据片段。

#### 分区表的开销为什么小？

> Partitioning has low overhead because **there is no data structure that points to rows** **and must be updated**—partitioning doesn’t identify data at the precision of rows, and has no data structure to speak of. Instead, it has an equation that says which partitions can contain which categories of rows.

因为不用像索引一样用特殊的数据结构来存储关于行的信息和指针，而且也不用更新这些信息和指针。

- 分区并没有把数据精确到行来区分，因此就没数据结构这回事。
- 只需要一个简单的表达式判断每个分区包含的是什么种类的行，这样在CURD时如果查询条件正好和这个表达式一致，就可以很轻易地知道需要的数据在哪个分区。
- 从锁的角度看，分区表可以像Java中的锁分段技术一样把巨大数据上加的单个锁变成多个锁，这样无疑减少了锁的互斥竞争，因此开销小。

### 如何使用分区表

1. 必须将查询需要扫描的分区个数限制在一个很小的数量。

   如果不这样做，基本上相当于没有分区。

2. 分区列和索引列需要匹配。

   如果定义的索引列和分区列不匹配，会导致查询无法进行分区过滤。因此分区的条件必须定义在被索引的列上去。而且MYSQL限制了主键必须是分区条件之一。

#### 具体操作

通常情况下可能会有两种操作：

- 把所有数据按照主键平分到每个分区表，但是要保证查询条件中使用主键查询：

  **HASH partitioning**：

  ```SQL
  CREATE TABLE t1 (
      id INT,
      year_col INT
  );
  ```

  This table can be partitioned by `HASH`, using the `id` column as the partitioning key, into 8 partitions by means of this statement:

  ```SQL
  ALTER TABLE t1
      PARTITION BY HASH(id)
      PARTITIONS 8;
  ```

  **RANGE partitioning**：

  ```SQL
  CREATE TABLE t1 (
      id INT,
      year_col INT
  )
  PARTITION BY RANGE (year_col) (
      PARTITION p0 VALUES LESS THAN (1991),
      PARTITION p1 VALUES LESS THAN (1995),
      PARTITION p2 VALUES LESS THAN (1999)
  );
  ```

- 把热点数据（最近的数据）分区存储，并保存在内存中以减少IO：



## Q&A

### like regexp 匹配时不区分大小写？

#### 首先厘清概念：

- 字符集（由`charset` 或 `char set`关键字指定）为字母和符号的集合；
- 编码为某个字符集成员的内部表示；
- 校对为规定字符如何比较的指令。

因此，**查询是否区分大小写与创建表时设定的被查询列的字符集的校对（`Collation`）有关。**

可以查询各等级的默认字符集和校对

```SQL
# 各种不同级别的默认字符集
show variables like 'character%';

# 各种不同级别的默认校对
show variables like 'collation%';
```

![1552356921773](images/mysql/1552356921773.png)

![1552356958259](images/mysql/1552356958259.png)

#### 实验验证：

按照下面创建表`account`：

```sql
create table account
(
  id int not null auto_increment,
  name varchar(50) binary not null,
  primary key (id)
);
```

在`name`列加上`binary`关键字约束后会直接改变这一列的字符集校对`COLLATION`。

观察结果就可以发现，虽然`name`列的字符集和默认字符集相同，但是`COLLATION`改变了：`utf8mb4_bin`是区分大小写的校对。

```SQL
> show create table account;

CREATE TABLE `account` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

下面插入数据并进行测试：

```MYSQL
insert into account (name)
values ('radon');
insert into account (name)
values ('Radon');

# 查询结果区分大小写
select name from account where name like 'radon';
select name FROM account WHERE name REGEXP 'radon';
```

![1552357215474](images/mysql/1552357215474.png)

```MYSQL
# 修改列定义，改为数据库级别的默认字符集和默认校对
alter table account modify name varchar(50) not null ;

# 查询建表，发现已经改为默认字符集和默认校对
show create table account;

CREATE TABLE `account` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

再次进行查询：

```MYSQL
# 查询结果不区分大小写
select name from account where name like 'radon';
select name FROM account WHERE name REGEXP 'radon';
```

![1552357499130](images/mysql/1552357499130.png)



### 表名和数据库名是否大小写敏感?

和下面两个全局变量有关

```MYSQL
show global variables like '%lower_case%';

+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_file_system | ON    |
| lower_case_table_names | 1     |
+------------------------+-------+
```

#### lower_case_file_system

表示当前系统文件是否大小写敏感，**只读参数，无法修改。**

ON  大小写不敏感 
OFF 大小写敏感 

#### lower_case_table_names

可以修改。

lower_case_table_names = 0时，mysql会直接按照操作系统是否区分大小写来操作。 

lower_case_table_names = 1时，mysql会先把表名转为小写，再按照操作系统是否区分大小写来操作。















