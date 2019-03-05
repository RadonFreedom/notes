## 分区表

### 为什么要使用分区表？

#### 有了索引为什么还用分区表？

参照 高性能Mysql 英文原文：

> It’s worth restating this: **at very large sizes, B-Tree indexes don’t work.** 
>
> - Unless the index covers the query completely, the server needs to look up the full rows in the table, and that causes random I/O a row at a time over a very large space, which will just kill query response times. 
> - The cost of maintaining the index (disk space, I/O operations) is also very high.



B+树索引将在超大数据量下失去作用，因为：

- 除非索引能正好满足查询条件，查询在得到索引返回的结果范围之后还是得全表扫描，这就造成了大量的随机IO（每次只在很大的一块数据区域内取出一条满足查询条件的数据，因此需要访问很多不同的磁盘块的一小点数据，而不是理想中的从少数磁盘块中取出查询到的大量数据），这将会造成让人无法接受的响应时间。
- 在数据量巨大的情况下，维护索引的开销也是非常高的。因为一级索引存储在内存中，很大的数据量将导致索引有很大的内存占用，更不用说其他索引所占用的磁盘空间和数据有变更时因为要同步修改索引带来的开销。