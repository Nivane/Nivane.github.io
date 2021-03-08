---
title:  "Doris Aggregate Model踩坑"
date:   2021-02-01 00:00:00
tags:
    - Doris
---


# Doris Aggregate Model踩坑
## 聚合模型的特性

* Doris不允许修改 Value 列的聚合类型(REPLACE/MIN/MAX/SUM)。

* Doris聚合模型添加列默认为Key，如果添加 Value, 需要指定聚合类型。


## 踩坑
**Doris Aggregate Model默认添加列类型为Key**

线上有一张表是 Aggregate Model, Value 类型包含了 SUM 和 REPLACE, 现在根据业务需求添加 Value 字段。
执行

```
alter table tmp add column col1 bigint;
```

因为 Aggregate Model 新建列默认是 Key 列，以上sql执行结束后col1列出现在了 Key 列，而不是期望的 Value 列。
为了解决这个问题，做了以下尝试：

1 是否可以 drop Key 列？
```
alter table tmp drop column col1;
```
执行失败：drop Key列时表中不允许存在REPLACE类型的列。

2 因为REPLACE类型的列在tmp表中无关紧要，是否可以drop REPLACE类型的 Value 后，drop Key列
```
alter table tmp drop column value_replace; // 执行成功
alter table tmp drop column col1; // 执行成功
```

3 重新建表刷数据
这个方法最直接，也最重。当然可以实现。

正确的添加 Value 列的方法是
```
alter table tmp add column col1 bigint sum comment "字段";
```


## help alter table
执行help alter table后，关于添加列的说明：

```
    1. 向 example_rollup_index 的 col1 后添加一个Key列 new_col(非聚合模型)
        ALTER TABLE example_db.my_table
        ADD COLUMN new_col INT Key DEFAULT "0" AFTER col1
        TO example_rollup_index;

    2. 向example_rollup_index的col1后添加一个value列new_col(非聚合模型)
          ALTER TABLE example_db.my_table
          ADD COLUMN new_col INT DEFAULT "0" AFTER col1
          TO example_rollup_index;

    3. 向example_rollup_index的col1后添加一个Key列new_col(聚合模型)
          ALTER TABLE example_db.my_table
          ADD COLUMN new_col INT DEFAULT "0" AFTER col1
          TO example_rollup_index;

    4. 向example_rollup_index的col1后添加一个value列new_col SUM聚合类型(聚合模型)
          ALTER TABLE example_db.my_table
          ADD COLUMN new_col INT SUM DEFAULT "0" AFTER col1
          TO example_rollup_index;

    5. 向 example_rollup_index 添加多列(聚合模型)
        ALTER TABLE example_db.my_table
        ADD COLUMN (col1 INT DEFAULT "1", col2 FLOAT SUM DEFAULT "2.3")
        TO example_rollup_index;
```