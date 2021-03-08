---
title:  "Spark SQL - Broadcast Join, Shuffle Hash Join, Sort Merge Join"
date:   2021-02-23 00:00:00
tags:
    - Spark
    - Join
---


# Spark SQL - Broadcast Join, Shuffle Hash Join, Sort Merge Join


## 1. 使用场景
1. Broadcast Hash Join

    大表和超小表

    spark.sql.autoBroadcastJoinThreshold 

    default 10M

2. Shuffle Hash Join

    大表和普通小表

3. Sort Merge Join

    大表和大表

    spark.sql.join.preferSortMergeJoin


**COST(Broadcast Hash Join) < COST(Shuffle Hash Join) < COST(Sort Merge Join)**

## 2. hash join
```sql
select a.* from a join b on a.id = b.id;
```
1. 小表作为Build Table，大表作为Probe Table。Build Table根据Join Key构建Hash Table，Probe Table根据Join Key进行探测，探测成功则执行join操作。
2. 构建的Hash Table最好能全部加载在内存，效率最高；内存的限制决定了hash join算法只适合至少一个小表的join场景，对于两个大表的join场景并不适用。

### 2.1 分布式中，HASH JOIN的两个变种
Hash Join适用于大表和小表Join，使用哪种变种需要参考小表有多大

1. Broadcast Hash Join
超小表使用广播
Spark SQL也可以根据内存资源、带宽资源适量将参数spark.sql.autoBroadcastJoinThreshold调大，让更多join执行broadcast hash join。

2. Shuffle Hash Join
一般小表使用shuffle hash join

## 3. Broadcast Hash Join (BHJ)

众所周知，在数据库通用模型（例如星型模型或雪花模型）中，表通常分为两种类型：事实表和维度表。维度表（小型表）通常是指固定的，可变性较小的表，例如联系人，项目等，一般数据是有限的。

事实表通常记录流水，如订单销售明细等，通常随着时间的增长而不断扩大，意味着大表。

当对维度表和事实表进行Join操作时，为了避免shuffle，我们可以将维度表的所有数据分发到每个执行节点供事实表使用。执行程序存储维度表的所有数据，在一定程度上牺牲了空间，替代很费时的shuffle操作，这在SparkSQL中称为Broadcast Join，如下所示：

![bhj](/images/3_common_join_spark_sql/bhj.png)
   
## 4. Shuffle Hash Join (SHJ)

![shj](/images/3_common_join_spark_sql/shj.png)
1.对两个表分别按照join key重新分区，即shuffle，目的是将具有相同join key的记录分配给相应的分区

2.将数据中的对应分区join，这里首先将小表分区构造为hash table，然后根据记录在join key中的大表中的键值取出来进行匹配。
   
## 5. Sort Merge Join (SMJ)

spark.sql.join.preferSortMergeJoin

SMJ适用于两张大表的join，两张大表join本身性能很重。

![smj](/images/3_common_join_spark_sql/smj.png)

* shuffle

两张大表都根据key进行重新分区，数据分布到各个节点

* sort

对每个分区节点的两个表的数据分别进行排序

* merge   

对排序的数据进行join操作。遍历两个有序表，碰到相同Join key就merge输出，否则从数据小的一遍继续遍历。


## 6. 参考引用

[1] https://sq.163yun.com/blog/article/155507707635548160

[2] https://www.linkedin.com/pulse/spark-sql-3-common-joins-explained-ram-ghadiyaram

[3] https://jaceklaskowski.gitbooks.io/mastering-spark-sql/content/spark-sql-SparkStrategy-JoinSelection.html







