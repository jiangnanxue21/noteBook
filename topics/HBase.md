# HBase

zk, 


分成以下几部分去总结：
1. 原始论文
2. HBase理论
3. 基于HBase的时序数据库opentsdb

## Bigtable

一旦开始分库分表了，就很难使用关系数据库的一系列的特性了。比如SQL里面的Join功能，或者是跨行的事务.Bigtable在一开始，也不准备先考虑事务、Join等高级的功能，而是把核心放在了“可伸缩性”上。因此，Bigtable自己的数据模型也特别简单，是一个很宽的稀疏表

![](Bigtable-1.png)

**对于列族，更合理的解读是，它是一张“物理表”，同一个列族下的数据会在物理上存储在一起。而整个表，是一张“逻辑表”**

Bigtable的开源实现HBase，就是把每一个列族的数据存储在同一个HFile文件里

**灵活、稀疏而又宽的表**，特别适合数据量很大，但是数据本身的Schema不确定的情况，加减字段都不需要停机或者锁表

### 动态区间分区

采用了一种自动去“分裂”（split）的方式来动态地进行分区

![](Bigtable-2.png)

整个数据表，会按照行键排好序，然后按照连续的行键一段段地分区。如果某一段行键的区间里，写的数据越来越多，占用的存储空间越来越大，那么整个系统会自动地将这个分区一分为二，变成两个分区。而如果某一个区间段的数据被删掉了很多，占用的空间越来越小了，那么我们就会自动把这个分区和它旁边的分区合并到一起

## opentsdb