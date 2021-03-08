---
title:  "Hive Spark Impala 高频参数配置"
date:   2020-10-24 00:00:00
tags:
    - Spark
    - Hive
    - Impala
---

# Hive Spark Impala 高频参数配置

## Hive


[https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties)

| 参数 |  |
| ----- |  |
| **hive.exec.dynamic.partition** | |
| **hive.exec.dynamic.partition.mode** | |
| **hive.exec.max.dynamic.partitions** | |
| **hive.exec.max.dynamic.partitions.pernode** |  |
| **hive.execution.engine** | |
| **hive.exec.max.created.files** |  |
| **hive.exec.parallel** |  |
| **hive.mapred.mode** |  |
| **hive.merge.mapredfiles** |  |
| **hive.optimize.sort.dynamic.partition** |  |
| **hive.auto.convert.join** | |
| **hive.auto.convert.join.noconditionaltask** | |
| **hive.compute.query.USING.stats** | |
| **hive.exec.dynamic.partition** | |
| **hive.exec.dynamic.partition.mode** | |
| **hive.exec.max.created.files** | |
| **hive.exec.max.dynamic.partitions** | |
| **hive.exec.parallel.thread.number** | |
| **hive.exec.reducers.bytes.per.reducer** | |
| **hive.fetch.task.conversion** | |
| **hive.fetch.task.conversion.threshold** | |
| **hive.groupby.skewindata** | |
| **hive.input.format** | |
| **hive.limit.pushdown.memory.usage** | |
| **hive.map.aggr** | |
| **hive.map.aggr.hash.percentmemory** | |
| **hive.merge.mapfiles** | |
| **hive.merge.mapredfiles** | |
| **hive.merge.size.per.task** | |
| **hive.merge.smallfiles.avgsize** | |
| **hive.merge.sparkfiles** | |
| **hive.optimize.bucketmapjoin.sortedmerge** | |
| **hive.optimize.index.filter** | |
| **hive.optimize.ppd** | |
| **hive.optimize.reducededuplication** | |
| **hive.optimize.reducededuplication.min.reducer** | |
| **hive.optimize.skewjoin** | |
| **hive.smbjoin.cache.rows** | |
| **hive.stats.autogather** | |
| **hive.stats.fetch.column.stats** | |

## Spark
[https://spark.apache.org/docs/latest/configuration.html](https://spark.apache.org/docs/latest/configuration.html)

| 参数 |  |
| ----- |  |
| **spark.master** | |
| **spark.serializer** | |
| **spark.sql.autoBroadcastJoinThreshold** | |
| **spark.default.parallelism** | | 
| **spark.executor.memory** | | 
| **spark.sql.adaptive.enabled** | | 
| **spark.sql.adaptive.shuffle.targetPostShuffleInputSize** | | 
| **spark.sql.crossJoin.enabled** | | 
| **spark.sql.shuffle.partitions** | | 
| **spark.driver.memory** | |
| **spark.dynamicAllocation.enabled** | |
| **spark.dynamicAllocation.maxExecutors** | |
| **spark.dynamicAllocation.minExecutors** | |
| **spark.executor.cores** | |
| **spark.executor.memoryOverhead** | |
| **spark.kryoserializer.buffer** | |
| **spark.kryoserializer.buffer.max** | |
| **spark.sql.shuffle.partitions** | |
| **spark.yarn.executor.memoryOverhead** | |

## Impala
[https://impala.apache.org/docs/build/html/topics/impala_query_options.html](https://impala.apache.org/docs/build/html/topics/impala_query_options.html)

| 参数 |  |
| ----- |  |
| **sync_ddl** | | 
| **mem_limit** | |

## MapReduce
[https://hadoop.apache.org/docs/r3.0.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml](https://hadoop.apache.org/docs/r3.0.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml)

| 参数 |  |
| ----- |  |
| **mapreduce.map.java.opts** | |
| **mapreduce.map.memory.mb** | |
| **mapreduce.reduce.memory.mb** | |
| **mapred.min.split.size** |  |
| **mapred.min.split.size.per.node** |  |
| **mapred.min.split.size.per.rack** |  |
| **mapreduce.fileoutputcommitter.marksuccessfuljobs** |  |
| **mapreduce.job.reduce.slowstart.completedmaps** |  |
| **mapred.job.name** | |
| **mapred.max.split.size** | |
| **mapreduce.reduce.java.opts** | |

## HDFS

| 参数 |  |
| ----- |  |
| **dfs.datanode.max.xcievers** | | 