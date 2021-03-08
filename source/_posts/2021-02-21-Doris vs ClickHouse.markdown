---
title:  "DorisDB vs ClickHouse"
date:   2021-02-21 00:00:00
tags:
    - Doris
    - ClickHouse
    - OLAP

---


# DorisDB vs ClickHouse


## 1. ClickHouse的缺点

### 1.1 过度依赖大宽表
* 构建和维护大宽表的工作量令人发指
* 无法利用星型模型和雪花模型

### 1.2 难以支撑高并发
* 高并发需要多集群？ 多集群意味着多备份，多运维

### 1.3 运维复杂度
* 依赖第三方的副本机制
* zk可能成为瓶颈


## 2. Doris建表的最佳实践
### 2.1 分区和分桶
![partition](/images/dorisdb_vs_clickhouse/partition.png)

Partition 是数据导⼊和备份恢复的最⼩逻辑单位

Tablet 是数据复制和均衡的最⼩物理单位

### 2.2 分区分桶与裁剪
![table_partition](/images/dorisdb_vs_clickhouse/table_partition.png)
![partition_tablet](/images/dorisdb_vs_clickhouse/partition_tablet.png)

* 针对 Doris ⽀持两层的数据划分

        1 第⼀层是 Partition 分区，⽀持 Range 的划分⽅式

        2 第⼆层是 Distribution 分桶，⽀持 Hash 的划分⽅式

* Partition 类似分表，是对⼀个表按照分区键进⾏切分，可以利⽤分区裁剪减少数据访问量，也可以根据
数据的**冷热程度**把数据划分到不同的介质上

* Distribution是对数据在不同BE上的数据分布⽅式，按照分桶键hash以后均匀分布在BE上，注意尽量选择分桶键让数据均衡，不要出现bucket数据倾斜的情况

* Bucket数量的需要适中，如果希望充分发挥性能可以设置为BE数量* CPU core数量 或者 BE数量* CPU core数量 / 2，最好tablet控制在100MB - 1GB之间，tablet太少并⾏度可能不够，太多可能元数据过多，底层scan并发太多性能下降。

### 2.3 排序键
![short_key](/images/dorisdb_vs_clickhouse/short_key.png)

* 排序键可以使⽤前缀索引（short key）和ZoneMap来进⾏⾼效的过滤，同时也能利⽤数据局部性来加速group by等聚合操作

* shortkey 的列只能是排序键的前缀, 列数不超过3, 字节数不超过36字节, 不包含FLOAT/DOUBLE类型的列,VARCHAR类型列只能出现⼀次, 并且是末尾位置

* 索引粒度1024 类似Clickhouse的index_granularity

* Zonemap 通过min max来快速过滤page中的数据


## 3. 整体架构
![architecture](/images/dorisdb_vs_clickhouse/architecture.png)

### 3.1 MPP架构
![mpp](/images/dorisdb_vs_clickhouse/mpp.png)
![mpp2](/images/dorisdb_vs_clickhouse/mpp2.png)

### 3.2 Profile
![profile](/images/dorisdb_vs_clickhouse/profile.png)
图形化的Profile 可以看到MPP的执⾏过程，⽅便的定位性能瓶颈和数据倾斜

**这个功能目前开源apache doris没有，只有dorisdb有**

## 4 计算引擎
### 4.1 向量化技术
![vectorization1](/images/dorisdb_vs_clickhouse/vectorization1.png)
![vectorization2](/images/dorisdb_vs_clickhouse/vectorization2.png)

* 向量化技术
内存结构按照列组织 引⼊Column结构，按列load数据，按列进⾏表达式计算和节点计算

* 优势
CPU 预取 / 分⽀预测友好 / Cache 友好 / ⽅便使⽤SIMD指令集

### 4.2 SIMD 指令
![SIMD](/images/dorisdb_vs_clickhouse/SIMD.png)

### 4.3 内存优化
![main_memory](/images/dorisdb_vs_clickhouse/main_memory.png)

### 4.4 Join优化
![join1](/images/dorisdb_vs_clickhouse/join1.png)
![join2](/images/dorisdb_vs_clickhouse/join2.png)

### 4.5 聚合优化
⾃适应两层聚合
![agg](/images/dorisdb_vs_clickhouse/agg.png)

* 分布式两阶段聚合

* Local Agg ：本地部分聚合

* Final Agg ： 全量聚合

* Steamming Vs TwoLevel Vs Suflfle By Column

* 运⾏时⾃适应

## 5 存储引擎

### 5.1 列式存储

![column](/images/dorisdb_vs_clickhouse/column.png)
![column2](/images/dorisdb_vs_clickhouse/column2.png)

### 5.2 前缀索引
![short](/images/dorisdb_vs_clickhouse/short.png)

### 5.3 Bitmap索引
![bitmap](/images/dorisdb_vs_clickhouse/bitmap.png)

### 5.4 延迟物化
![late_materialization](/images/dorisdb_vs_clickhouse/late_materialization.png)

### 5.5 向量化过滤
![vectorization3](/images/dorisdb_vs_clickhouse/vectorization3.png)

### 5.6 低基数优化字典编码
![low_cardinality](/images/dorisdb_vs_clickhouse/low_cardinality.png)

## 6 极速MPP
![the_mpp](/images/dorisdb_vs_clickhouse/the_mpp.png)

## 7 除了快，还有什么
### 7.1 使用简单
* Mysql协议兼容
    * 标准SQL，兼容各类主流BI，迁移成本极低
    * 函数⽀持丰富，包括各类时间、字符串、GIS、聚合函数以及窗⼝函数
* 灵活的数据模型
    * ⾼效⽀持星型/雪花模型，避免打平成⼤宽表，更加灵活的适应实时更新和维度变化
    * Duplicate/Unique/Aggregate三种模式适配各类场景
* 开箱即⽤的⽤户体验
    * ⾃适应应对各种case，⽆须额外的复杂优化⼿段
    * 各种Load⼿段可以降低ETL⼯作量，通过配置快速导⼊数据
    
### 7.2 运维方便
* ⾼可⽤
    * FE采取类Paxos⼀致性算法，⽆单点瓶颈
    * BE多副本可以独⽴调节，数据按⾃身特性灵活拆分
* ⾼度⾃治
    * ⽆外部依赖，避免繁杂的中间件维护
    * 副本⾃修复，数据⾃均衡，⼀键扩容，缩容
* 可视化管理⼯具
    * 全局配置，⼀键部署，滚动升级
    * 完备的监控告警集成保障系统稳定可观测
    * 灵活的扩容缩容⽅便集群容量规划

### 7.3 生态完善
![sys](/images/dorisdb_vs_clickhouse/sys.png)

### 7.4 稳定可靠
* 完善的测试
    * 完成SSB TPCH TPCDS NYC-TAXI 等测试集功能完备
    * SQLancer 每天构建数千万条随机SQL验证
    * 引⼊SQLite/Spark等外部DB 200万标准测试集通过
* ⾼可⽤架构
    * 没有单点，FE ⾼可⽤，BE多副本
    
### 7.5 统一分析平台
![general](/images/dorisdb_vs_clickhouse/general.png)

## 8 DorisDB Vs. ClickHouse

|  | ClickHouse | DorisDB |
| ----- | ---- | ---- |
| **标准SQL语⾔的⽀持** | 不⽀持标准的SQL语⾔，⽆法直接对接主流的BI系统。 |⽀持标准的SQL语⾔，兼容MySQL协议，可以直接对接主流的BI系统。 |   
| **分布式Join** | ⼏乎不⽀持分布式Join，在分析模型上仅⽀持⼤宽表模式。 | ⽀持各种主流分布式Join，不仅⽀持⼤宽表模型，还⽀持星型模型和雪花模型。|
| **⾼并发查询** | 传统MMP数据分布⽅式，⼩查询会极⼤消耗集群的资源，⽆法实现⾼并发查询，并且⽆法通过扩容的⽅式来提⾼并发能⼒。 | 现代化MPP数据分布⽅式，数据按照分⽚的⽅式保存，⼩查询只需要⽤到部分机器资源，能极⼤地提⾼并发查询量。 | 
| **MPP架构** | Scatter-Gather 模式，聚合操作依赖单点完成，操作数据量⼤时存在明显的性能瓶颈。 | 现代化MPP架构，可以实现多层聚合，能够执⾏复杂的SQL查询，⼤表Join，⾼基数聚合查询等。 |  
| **精确去重** | 对⾼基数列进⾏精确去重操作（如计算APP的DAU）时，受限于单点聚合的处理⽅式，性能瓶颈明显。 | 现代化MPP架构，可以实现多层聚合，能有效利⽤多机资源，保证查询性能。
| **Exactly once语义** | 数据导⼊⽆事务保证，⽆法保证数据写⼊的“不丢不重”，订单类场景⽆法使⽤。 | 数据导⼊有事务保证，可以很容易地实现Exactly once语义，数据导⼊“不丢不重”。 
| **集群扩容** | 传统MPP数据分布⽅式，数据扩容时需要进⾏数据重分布，需要⼈⼯操作，⼯作量巨⼤，影响线上服务。 | 现代化MPP数据分布⽅式，扩容时只需要迁移部分数据分⽚⾛即可，系统⾃动完成，不影响线上服务。
| **运维** | 分布式⽅案依赖Zookeeper，在集群扩⼤时，Zookeeper会变成性能瓶颈，额外运维和维护成本⾼。 | 不依赖任何外部系统，整个系统只有两种进程，⾃动故障恢复，极简运维。 |
| **社区⽣态** | 整个开源社区被俄罗斯公司把持，在中国没有商业化公司⽀持，使⽤上规模后技术⽀持⽆法保证。 | 开源社区的核⼼研发⼈员都是中国⼈，在国内有商业化公司⽀持，服务更加本地化，技术⽀持⽆障碍。 |

## 9 参考引用
[1] https://zhuanlan.zhihu.com/p/344031483

[2] DorisDB 技术分享会PPT








