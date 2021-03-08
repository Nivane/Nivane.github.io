---
title:  "Spark SQL - Join Types"
date:   2021-02-23 00:00:00
tags:
    - Spark
    - Join
---


# Spark SQL - Join Types


## 1. 常见的SQL JOIN

![joins](/images/join_types/joins.png)


| SQL | Name	 | JoinType |
| ----- | ---- | ---- |
| **CROSS** |  cross |CROSS |   
| **INNER** | inner | INNER |
| **FULL OUTER** | outer, full, full outer | FULL OUTER | 
| **LEFT ANTI** | left anti, anti | LEFT ANTI |  
| **LEFT OUTER** | left outer, left | LEFT OUTER |
| **LEFT SEMI** | left semi, semi | LEFT SEMI |
| **RIGHT OUTER** | right outer, right | RIGHT OUTER |
| **NATURAL** | Special case for INNER, LEFT OUTER, RIGHT OUTER, FULL OUTER | NATURAL |
| **USING** | Special case for INNER, LEFT OUTER, LEFT SEMI, RIGHT OUTER, FULL OUTER, LEFT ANTI| USING |


## 2. left semi join

当JOIN条件成立时，返回左表中的数据。如果mytable1中某行的id在mytable2的所有id中出现过，则此行保留在结果集中。

```sql
SELECT * FROM mytable1 a LEFT SEMI JOIN mytable2 b ON a.id=b.id;
```
**只会返回mytable1中的数据**，只要mytable1的id在mytable2的id中出现。

## 3. left anti join

当JOIN条件不成立时，返回左表中的数据。如果mytable1中某行的id在mytable2的所有id中没有出现过，则此行保留在结果集中。

```sql
SELECT * FROM mytable1 a LEFT ANTI JOIN mytable2 b ON a.id=b.id;
```
**只会返回mytable1中的数据**，只要mytable1的id在mytable2的id没有出现。


## 4. 参考引用

[1] https://jaceklaskowski.gitbooks.io/mastering-spark-sql/content/spark-sql-joins.html

[2] https://stackoverflow.com/questions/45990633/what-are-the-various-join-types-in-spark/45990634

[3] https://www.alibabacloud.com/help/zh/doc-detail/73784.htm

[4] https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-hints.html

[5] https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-join.html

[6] https://help.aliyun.com/document_detail/73784.html









