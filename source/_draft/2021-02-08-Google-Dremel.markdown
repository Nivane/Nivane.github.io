---
title:  "Google Dremel论文翻译"
date:   2021-02-08 00:00:00
tags:
    - Dremel
    - Google
    - 列式存储
---



# Google Dremel
## 摘要

Dremel 是可扩展，交互式的 ad-hoc 查询系统，用于只读嵌套数据的分析。通过整合 multi-level execution tree 和列式存储，能够实现万亿行表
在数秒内聚合查询。在谷歌，Dremel 扩展到了数以千计的 CPU 和 PB 级别的数据。在本论文中，我们描述了 Dremel 的体系结构和实现，解释了它如何
实现基于MapReduce计算。我们展示了嵌套记录的一种新型列存储形式，并讨论基于该系统几千个节点实例的实验。

## 1. 介绍

大规模的分析数据处理已经在网络公司和跨行业中广泛应用，这主要是因为低成本的存储能够收集大量的关键业务数据。将这些数据放在分析师和工程师的
指尖变得越来越重要；交互响应时间通常在数据探索、监控、在线客户支持、快速原型设计、数据管道调试和其他任务中起到本质作用。执行大规模交互式
数据分析需要高并行度。例如，想要使用今天的商用磁盘在一秒钟内读取 1TB 的压缩数据，需要数万个磁盘并行读取。类似地，CPU密集型查询可能需要在
数千个内核上运行才能在几秒钟内完成。在谷歌，大规模并行计算是使用商用机器的共享集群来完成的[5]。集群通常承载大量的分布式应用程序，这些应用
程序共享资源，具有广泛变化的工作负载，并且在具有不同硬件参数的计算机上运行。分布式应用程序中的单个工作进程执行给定任务的时间可能比其他工
作进程长得多，或者可能由于集群管理系统的故障或抢占而永远无法完成。因此，处理掉队和故障对于实现快速执行和容错至关重要[10]。

网络和科学计算中使用的数据通常是不相关的。因此，在这些领域中，灵活的数据模型至关重要。编程语言中使用的数据结构，分布式系统交换的消息，结
构化文档等，自然地就能使自己适合于嵌套表示。但在 web scale 标准化和重新组合此类数据通常是禁止的。嵌套数据模型是Google [21]和其他主要网
络公司的大多数结构化数据处理的基础。本文介绍了一个名为 Dremel 的系统，该系统支持在共享的商用机器集群上对超大型数据集进行交互式分析。与传
统数据库不同，它能够处理原位嵌套数据。原位是指“就地”访问数据的能力，例如在分布式文件系统（例如GFS [14]）或另一个存储层（例如Bigtable 
[8]）中。 Dremel可以对此类数据执行多种查询，这些查询通常需要一系列MapReduce（MR [12]）作业，但执行时间只是MapReduce的一小部分。 
Dremel不是为了替代MR，Dremel 经常与 MR 结合使用以分析MR pipelines 的输出, 或者快速原型化较大的计算。

Dremel 自2006年起投入生产，在Google中拥有数千用户。 公司部署了多个Dremel实例，从数十个节点到数千个节点。 使用该系统的业务和场景包括：

* 分析爬虫Web文档。
* 跟踪Android Market中应用程序的安装数据。
* Google产品的崩溃报告。
* Google图书的OCR结果。
* 垃圾邮件分析。
* 在Google Maps上调试地图图块。
* Bigtable实例中的Tablet迁移。
* 在Google的分布式构建系统上运行的测试结果。
* 数十万个磁盘的磁盘I/O统计信息。
* 对作业的资源监控在Google的数据中心中运行。
* Google代码库中的符号和依赖项。

Dremel 建立在Web搜索和并行DBMS的思想基础上。 首先，其架构借鉴了分布式搜索引擎中使用的服务树的概念[11]。 就像网络搜索请求一样，查询被推
入树中，并在每个步骤被重写。 查询的结果是通过汇总从下级树收到的答复来组合的。 其次，Dremel提供了一种类似于SQL的高级语言来表达即席查询。 
与诸如Pig [18]和Hive [16]之类的层相反，它本地执行查询，而无需将其转换为MR作业。

// 关系数据和嵌套数据是两种数据模型？
最后，重要的是，Dremel使用了列式存储表示形式，这使它能够从辅助存储中读取较少的数据，并由于廉价的压缩而降低了CPU成本。列存储已被用于分析
关系数据[1]，但就我们所知，还没有扩展到嵌套数据模型。 Google提供的许多数据处理工具都支持我们提出的列式存储格式，包括MR，Sawzall [20]
和FlumeJava [7]。在本文中，我们做出了以下贡献：

* 我们描述了一种用于嵌套数据的新型列式存储格式。我们提出了将嵌套记录分解为列并重新组装它们的算法（第4节）。
* 我们概述了Dremel的查询语言和执行方式。两者都旨在对列式嵌套数据进行有效操作，并且不需要重组嵌套记录（第5节）。
* 我们展示了如何将Web搜索系统中使用的执行树应用于数据库处理，并说明它们在高效聚合查询方面的好处（第6节）。
* 我们介绍了在1000-4000个节点运行的系统实例上进行的trillion记录，多TB数据集的实验（第7节）。

此篇文章的结构如下。在第2节中，我们将说明如何将Dremel与其他数据管理工具一起用于数据分析。第3节介绍了其数据模型。第4-8节介绍了上面列出的
主要贡献。相关工作在第9节中讨论。第10节是结论。

## 2.  背景

我们首先从一个场景开始，该场景说明了交互式查询处理如何适应更广泛的数据管理生态系统。假设Google的工程师爱丽丝（Alice）提出了一个新颖的想
法从网页中提取各种信号。她运行MR作业，处理输入数据，并生成包含新信号的数据集，该数据集存储在分布式文件系统中的数十亿条记录中。为了分析她
的实验结果，她启动了Dremel并执行了几个交互式命令：
```
DEFINE TABLE t AS /path/to/data/* 
SELECT TOP（signal1，100），COUNT（*）FROM t
```
她的命令在几秒钟内执行。她还进行了其他一些查询，以说服自己算法有效。她发现了signal1中的不规则之处，并通过编写FlumeJava [7]程序对它的输
出数据集进行了更复杂的分析计算，从而对它进行了更深入的研究。解决问题后，她将建立一个流水线来连续处理传入的输入数据。她制定了一些固定的SQL
查询，以汇总其各个维度的流水线结果，并将其添加到交互式仪表板中。最后，她将自己的新数据集注册到目录中，以便其他工程师可以快速找到并查询它。

 以上情形要求查询处理器和其他数据管理工具之间的互操作。 为此的第一个要素是公用存储层。 Google文件系统（GFS [14]）是公司中广泛使用
 的一种此类分布式存储层。GFS使用副本来保存数据，防止硬件故障导致数据问题，并且在出现落后情况时获得快速响应时间。 高性能存储层对于原位数
 据管理至关重要。 高性能存储允许在没有load阶段时间消耗的情况下访问数据，这是分析数据处理中数据库使用的主要障碍[13]，在DBMS能够加载数据
 和执行一个查询之前，通常要跑一些MR分析。 另外一个好处是，可以使用标准工具方便地操作文件系统中的数据，例如，以传输到另一个群集，更改访问
 权限或基于文件名识别数据子集进行分析。
 
 ![column-oriented](source/images/dremel/column-oriented.jpg)
 
 
 构建可互操作的数据管理组件的第二个要素是共享存储格式。事实证明，列式存储对于关系数据是成功的，但要使其适用于Google，则需要对其进行调整
 以适应嵌套数据模型。图1说明了主要思想：嵌套字段（例如A.B.C）的所有值都连续存储。 因此，无需阅读A.E，A.B.D等就可以检索A.B.C。我们要解
 决的难题是如何保留所有结构信息并能够从任意字段子集重建记录。 接下来，我们讨论数据模型，然后转向讨论算法和查询处理。
 
 ## 3. 数据模型
 
 在本节中，我们介绍Dremel的数据模型，并介绍稍后使用的一些术语。 数据模型起源于分布式系统的上下文（解释其名称为“Protocol Buffers” [21]
 ），已在Google广泛使用，并且可以作为开源实现。 数据模型基于强类型的嵌套记录。 它的抽象语法如下：
![data-model-formula](source/images/dremel/formula.jpg)

其中 &tau; 是原子类型或记录类型。 dom中的原子类型包括整数，浮点数，字符串等。记录由一个或多个字段组成。 记录中的字段i具有名称Ai和可选的
多重性标签。 *Repeated*的字段（\*）在一条记录中可能多次出现。 它们被解释为值列表，即记录中字段出现的顺序很重要。 记录中可能缺少Optional
字段（？）。 否则，字段是必填字段，即必须仅出现一次。  

为了进行说明，请看图2。图2描述了定义记录类型Document的schema，该记录类型表示Web文档。 schema定义使用[21]中的具体语法。 文档具有
Required类型的整数DocId和Optional类型的Links，Links包含Forward和Backward entries的列表，Forward和BackWard中包含了前置和后续页面
 的DocId，一个Document可以有多个Names，可以引用不同URL。 Name包含一系列Code和（Optional) Country 对。 图2还显示了两个符合schema的
 示例记录 r1和 r2。使用缩进来概述记录结构。 我们将使用这些sample记录在下一节来解释算法。 模式中定义的字段形成树层次结构。嵌套字段的完整
 路径使用通常的点分符号表示，例如Name.Language.Code。 

![schema](source/images/dremel/schema.jpg)

在Google，这种嵌套数据模型支持与平台无关可扩展的机制，用于序列化结构化数据。 代码生成工具为诸如C ++或Java的编程语言生成绑定。 通过使用
记录的标准二进制在线表示可实现跨语言的互操作性，其中字段值在记录中出现时按顺序进行排列。 这样，用Java编写的MR程序可以使用来自通过C++库暴
露的数据源的记录。因此，如果记录以列式存储，则快速组装记录对于与MR以及其他数据处理工具的互操作非常重要。

## 4. 嵌套列式存储

如图1所示，我们的目标是连续存储给定字段的所有值以提高检索效率。在本节中，我们解决以下挑战：以列式无损表示记录结构（第4.1节），快速编码（
第4.2节）和有效的记录整合（第4.3节）。

### 4.1 重复和定义level

单独的值不能表达记录的结构。给定Repeated字段的两个值，我们不知道重复的值在哪个“level”（例如，这些值是来自两个不同的记录，还是来自同一记
录中的两个重复的值）。同样，在缺少Optional字段的情况下，我们也不知道显式定义了哪些封闭记录。因此，我们介绍了repetition和definition 
level 的概念，这些概念在接下来定义。 为了参考，请看图3，该图总结了sample记录中所有atomic 字段的repetition 和 definition level。

**Repetition Level**。查看图2中的字段Code。它在r1中出现3次。 “ en-us”和“ en”出现在第一个Name中，而“en-gb”出现在第三个Name中。为了消除
这些出现的歧义，我们将Repetition Level附加到每个值。它告诉我们该值在字段路径中的哪个Repeated 字段重复。字段路径Name.Language.Code包
含两个重复的字段，即Name和Language。因此，Code的重复级别在0到2之间；级别0表示新Record的开始。现在假设我们从上向下扫描记录r1。当我们遇到
“en-us”时，我们没有看到任何重复的字段，即repetition level为0。当我们看到“en”时，字段Language已重复，因此repetition level为2。最后，
当我们遇到“en-gb'，Name最近一次重复（Language仅在Name之后出现一次），因此repetition level 为1。因此，r1中Code值的repetition level为0、2、1。请注
意，r1中的第二个Name不包含任何Code值。为了确定“en-gb”出现在第三个Name中而不是第二个Name中，我们在“en”和“en-gb”之间添加了一个NULL值（
参见图3）。代码是“Language”中的必填字段，因此缺少Code这一事实意味着未定义Language。通常，确定嵌套记录的级别需要额外的信息。
 
[]https://stackoverflow.com/questions/43568132/dremel-repetition-and-definition-level
[] https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html


level 0 denotes the start of a new record.


 **Definition levels**. Each value of a field with path p, esp. every NULL, has a definition level specifying how many 
 fields in p that could be undefined (because they are optional or repeated) are actually present in the record. 
 To illustrate, observe that r1 has no Backward links. However, field Links is defined (at level 1). To preserve this 
 information, we add a NULL value with definition level 1 to the Links.Backward column. Similarly, the missing 
 occurrence of Name.Language.Country in r2 carries a definition level 1, while its missing occurrences in r1 have 
 definition levels 2 (inside Name.Language) and 1 (inside Name), respectively. 
 
 We use integer definition levels as opposed to is-null bits so that the data for a leaf field (e.g., Name.Language.Country) contains the information 
 about the occurrences of its parent fields; an example of how this information is used is given in Section 4.3. The 
 encoding outlined above preserves the record structure losslessly. We omit the proof for space reasons.
 
**编码**。每列存储为一组blocks。每个block包含repetition和definition level（以下简称为级别）和压缩过的字段值。NULL不是显式存储的，因
为是由definition level确定：任何definition level小于字段路径中重复字段和可选字段的数量，则表示NULL。始终定义的值不会存储definition
 level。 同样，仅在需要时才存储repetition level。 例如，definition level0表示repetition level 0，因此可以省略后者。实际上，在图3
 中，没有为DocId存储任何level。 级别打包为bit sequence。 我们只使用必要的位数； 例如，如果最大definition level为3，则每个
 definition level使用2位。
 
 ### 4.2 记录拆分为列
 
 上面我们以列格式展示了记录结构的编码。我们要解决的下一个挑战是如何高效生产具有repetition和definition level的column stripe。
 
 附录A中给出了计算repetition和definition level的基本算法。该算法递归到记录结构中，并为每个字段值计算级别。如前所述，即使缺少字段值，
 也可能需要计算repetition和definition level。 Google使用的许多数据集都是稀疏数据集。包含数千个字段的schema并不少见，很多记录仅使用了
 其中的近百个字段。因此，我们尝试尽可能不花大精力去处理缺失字段。为了产生column stripe，我们创建了一个field writers树，其结构与
 schema的字段层次结构相匹配。基本思想是仅在field writers拥有自己的数据时才对其进行更新，除非绝对必要，否则不要尝试将父状态传播到树中。
 为此，child writers从parent那里继承level。每当添加新值时，a child writer都会与其parent's levels同步。
 
 ### 4.3 Record Assembly
 
从列式数据有效地组合记录，对于面向记录的数据处理工具（例如MR）至关重要。给定字段的子集，我们的目标是重建原始记录，就好像它们只包含所选字段
一样，其他所有字段都被删除。关键思想是这样的：我们创建一个有限状态机（FSM），以读取每个字段的字段值和级别，并将这些值顺序地附加到输出记
录中。 **FSM状态对应一个用于每个选定字段的field reader**。状态转换用repetition level标记。reader获取值后，我们将查看下一个
repetition level，以确定要使用的下一个reader。对于每个记录，从开始状态到结束状态都要遍历FSM一次。图4显示了一个FSM，它在我们的运行示例
中重建了完整的记录。起始状态为DocId。读取DocId值后，FSM将转换为Links.Backward。清除所有重复的Backward值之后，FSM跳至Links.Forward
等。记录组合算法的详细信息在附录B中。要勾勒出如何构造FSM转换，让l为当前对象返回的下一个repetition level。当前field reader f。从
schema tree中的f开始，我们找到其祖先，该祖先在级别l处重复，然后选择该祖先内的第一个叶子字段n。这给我们提供了FSM转换（f; l）！ 。例如，
令l = 1为f = Name.Language.Country读取的下一个重复级别。其重复级别为1的祖先是Name，其第一个叶子字段是n = Name.Url。 FSM构造算法的
详细信息在附录C中。如果仅需要检索字段的子集，我们将构造一个更简单且更便宜的FSM。图5描述了一个FSM，用于读取DocId和
Name.Language.Country字段。该图显示了自动机生成的输出记录s1和s2。请注意，我们的编码和汇编算法保留了Country字段的封闭结构。这对于需要
访问的应用程序非常重要，例如，以第二种姓氏的第一种语言显示的国家/地区。在XPath中，这对应于评估表达式的能力，例如
/Name[2]/Language[1]/Country。


## 5. 查询语言

Dremel的查询语言基于SQL，旨在在列式嵌套存储中高效实现。正式定义语言不在本文讨论范围之内。相反，我们说明它的风格。每个SQL语句（及其转换
成的代数算子）都将一个或多个嵌套table及其schema作为输入，并生成一个嵌套表及其输出schema。图6描述了执行projection，selection和记录内
aggregation的示例查询。在图2中的表t = {r1, r2}上评估查询。这些字段是使用路径表达式引用的。尽管查询中没有记录构造函数，但查询会产生嵌套
结果。

为了解释查询的作用，请考虑选择操作（WHERE子句）。可以将嵌套记录视为带标签的树，其中每个标签都对应一个字段名称。select operator修剪掉不
满足指定条件的树的分支。因此，仅保留那些嵌套的记录，其中定义了Name.Url并以http开头。接下来，考虑projection。 SELECT子句中的每个标量表
达式在嵌套级别上都使用与该表达式中使用次数最多的输入字段相同的值。因此，字符串连接表达式在输入模式中以Name.Language.Code级别发出Str值。
COUNT表达式说明了记录内聚合。聚合是在每个Name子记录内完成的，并以非负64位整数（uint64）的形式发出每个Name的Name.Language.Code出现的
次数。该语言支持嵌套子查询，记录间和记录内聚合，top-k，联接，UDF等；实验部分举例说明了其中一些功能。 
 

## 6. 查询执行

为了简单起见，我们在讨论在只读系统情境下的核心思想。许多Dremel查询是one-pass聚合；因此，我们将重点说明这些内容，并在下一部分中将其用于
实验。我们将对连接，索引，更新等的讨论推迟到以后的工作中。

*树结构*。 Dremel使用多层serving tree执行查询（请参见图7）。根server接收传入的查询，从表中读取元数据，并将查询路由到serving tree中
的下一个级别。叶子server与存储层通信或访问本地磁盘上的数据。考虑下面的一个简单的聚合查询：
```sql
SELECT A, COUNT(B) from T GROUP BY A
```
当root server收到上述查询时，它将确定包含T的所有 **tablets** ，即表的水平分区，并按如下方式重写查询：从表R1 1中选择（R1 1 UNION ALL ... R1n）GROUP A的SUM（c）； R1n是发送到节点1的查询结果； ：：：;服务树第1级的n：


*查询调度器*。 Dremel是一个多用户系统，即通常会同时执行几个查询。查询调度器根据查询的优先级调度查询并平衡负载。它的另一个重要作用是当一
台服务器变得比其他服务器慢或无法访问副本tablet时，能够提供容错能力。每个查询处理的数据量通常大于可用于执行的processing units，我们称其
为插槽（slot）。插槽与leaf server上的执行线程相对应。例如，一个由3,000个leaf server组成的系统，如果每个server使用8个线程，则具有
24,000个插槽。因此，可以通过为每个插槽分配大约5个tablets来处理一张100,000tablets的表。在查询执行期间，查询调度程序将计算tablets处理
时间的直方图。如果tablets要花费不成比例的长时间(long time)来处理，则会在另一台server上重新安排其时间。有些tablets可能需要重新调度多次
。leaf server以列​​表示形式读取嵌套数据的stripe。每个条带中的块都是异步预取的。预读缓存的命中率通常为95％。tablets通常是三向复制。当
leaf server无法访问一个tablet副本时，它将转向访问另一个副本。

查询调度程序接受一个参数，该参数指定返回结果之前必须扫描的tablets的最小百分比。正如我们将演示的那样，将此类参数设置为较低的值（例如98％
而不是100％）通常可以显着加快执行速度，尤其是在使用较小的复制因子时。每个server都有一个内部执行树，如图7右侧所示。内部树对应于物理查询执
行计划，包括标量表达式的求值。(codegen)为大多数标量函数生成优化的，特定类型的代码。project-select-aggregate查询的执行计划由一组迭代
器组成，这些迭代器以锁步的方式扫描输入列，并产生聚合和标​​量函数的结果，标注正确的重复和定义级别，从而在查询执行期间完全绕过record 
assembly。有关详细信息，请参阅附录D。一些Dremel查询（例如top-k和count-distinct）使用已知的one-pass算法（例如[4]）返回近似结果。

## 7. 实验

在本部分中，我们将评估Dremel在Google使用的多个数据集上的性能，并校验列式存储嵌套数据的高效性。我们的研究中使用的数据集的属性总结在图8中
。在未压缩，未复制的形式中，数据集占据了大约PB的空间。所有表都是三副本(three-way replicated)的，一个两副本(two-way replicated)的表
除外，并且包含100K至800K大小不等的tablets。我们首先检查单台计算机上的基本数据访问特征，然后说明列式存储如何使MR执行高效，最后集中讨论
Dremel的性能。实验是在常规业务运营期间，在与许多其他应用程序相邻的两个数据中心中运行的系统实例上进行的。除非另有说明，执行时间是五次运行
的平均时间。下面使用的表和字段名称是匿名的。

*本地磁盘*。在第一个实验中，我们检查了列式存储与面向记录的存储的性能折衷，扫描了包含大约300K行的表T1的1GB片段（请参见图9）。数据存储在
本地磁盘上，以压缩的列表示形式占用约375MB。面向记录的格式使用较大的压缩率，但在磁盘上产生的大小大致相同。该实验是在具有可提供70MB/s读取
带宽的磁盘的双核Intel机器上完成的。All reported times are cold; 所有时间都是cold；每次扫描之前，都会清除操作系统缓存。

该图显示了五个图形，说明对字段的子集做读取数据，解压数据，组合记录和解析记录所花费的时间。图（a）-（c）概述了列式存储的结果。这些图表中的
每个数据point均是通过对30个运行进行平均计算而获得的，在每个运行中，随机选择了一组给定基数的列。图（a）显示了读取和减压时间。图（b）增加
了从列组合嵌套记录所需的时间。图（c）显示了将记录解析为强类型的C++数据结构需要多长时间。图（d）-（e）描述了访问面向记录的存储中的数据的
时间。图（d）显示了读取和解压时间。大部分时间都花在解压上；实际上，压缩数据可以在大约一半的时间内从磁盘读取。如图（e）所示，解析在读取和
解压缩时间之外又增加了50％。这些消耗将paid给所有字段，包括不需要的字段。该实验的主要内容如下：当读取很少的列时，列式存储的效益大约是一个
数量级。列状嵌套数据的检索时间随字段数线性增长。记录的组合和解析是昂贵的，每个过程都可能使执行时间加倍。我们在其他数据集上观察到了类似的
趋势。一个自然的问题是，顶部和底部的图形在哪里交叉，即按记录存储的性能开始超过列式存储。根据我们的经验，交叉点通常位于数十个字段，但是交
叉点在数据集中会有所不同，并取决于是否需要做record assembly。

*MR和Dremel*。接下来，我们说明对比列数据和面向记录的数据的MR和Dremel执行。我们考虑访问单个字段的情况，即性能提升最为明显的场景。可以使
用图9的结果来推断多列的执行时间。在本实验中，我们计算表T1的txtField字段中的平均terms数。 MR执行是通过以下Sawzall [20]程序完成的:
```
numRecs: table sum of int;
numWords: table sum of int;
emit numRecs <- 1;
emit numWords <- CountWords(input.txtField);
```
记录数存储在变量numRecs中。对于每个记录，numWords都将由CountWords函数返回的input.txtField中的term数量增加。程序运行后，平均词频可以
计算为numWords = numRecs。在SQL中，此计算表示为:
```
Q1: SELECT SUM(CountWords(txtField)) / COUNT(*) FROM T1
```

图10以对数显示两个MR作业和Dremel的执行时间。两个MR jobs均由3000workers执行。同样，一个3000节点的Dremel实例用于执行查询Q1。 Dremel
和MR-on-column读取大约0.5TB的压缩列数据，而MR-on-Records读取为87TB。如图所示，通过从面向记录的存储切换到列式存储（从数小时到数分钟）
，MR的效率提高了一个数量级。使用Dremel（从几分钟到几秒钟）可以达到另一个数量级。

*serving tree拓扑*。 在下一个实验中，我们将展示服务树深度对查询执行时间的影响。 我们考虑对表T2的两个GROUP BY查询，每个查询都是通过对
数据的一次扫描来执行的。 表T2包含240亿个嵌套记录。 每个记录都有一个重复的字段项目，其中包含数字。 字段item.amount在数据集中重复约400亿
次。 第一个查询按国家汇总项目数量：

```sql
Q2: SELECT country, SUM(item.amount) FROM T2
GROUP BY country
```
它返回几百条记录，并从磁盘读取大约60GB的压缩数据。 第二个查询在具有选择条件的文本字段domain上执行GROUP BY。 它读取约180GB，并产生约
110万个不同的domain：
```sql
Q3: SELECT domain, SUM(item.amount) FROM T2
WHERE domain CONTAINS '.net'
GROUP BY domain
```

图11显示了作为服务器拓扑函数的每个查询的执行时间。 在每种拓扑中，leaf server的数量保持恒定在2900，因此我们可以假定相同的累积扫描速度。
在2级拓扑（1：2900）中，单个root server直接与leaf server通信。 对于3个级别，我们使用1：100：2900的设置，即额外设置100个中间服务器。
4级拓扑是1：10：100：2900。

当在服务树中使用3个级别时，查询Q2将在3秒内运行，并且从额外级别中受益不多。相反，由于增加了并行度，Q3的执行时间减少了一半。 在第2层，由于
root server需要按顺序汇总从数千个节点接收到的结果，因此Q3不在图表之列。 此实验说明了返回许多组的聚合如何从多级服务树中受益。


*Pertablet 直方图*。为了更深入地了解查询执行过程中发生的情况，请查看图12。该图显示了针对特定的Q2和Q3运行，leaf server处理tablets有多
快。 从tablet计划在可用插槽中执行的时间点开始测量时间，即不包括在作业队列中等待的时间。 此度量方法论排除了同时执行其他查询的影响。 每个
直方图下的面积对应于100％。 如图所示，一秒钟（或两秒钟）内处理了99％的Q2（或Q3）tablets。

*记录内聚合*。 作为另一个实验，我们检查了在表T3上运行的查询Q4的性能。该查询说明了记录内聚合：它对a.b.c.d值之和大于
a.b.p.q.r值之和的所有记录进行计数。字段在不同的嵌套级别重复。 由于列条带化，仅从磁盘读取了13GB（70TB中的），并且查询在15秒内完成。如果
不支持嵌套，那么在T3上运行此查询将非常昂贵。
```sql
Q4 : SELECT COUNT(c1 > c2) FROM
(SELECT SUM(a.b.c.d) WITHIN RECORD AS c1,
SUM(a.b.p.q.r) WITHIN RECORD AS c2
          FROM T3)
```

*可扩展性*。 以下实验说明了万亿记录表上系统的可伸缩性。 下面显示的查询Q5在表T4中选择了排名前20位的aid及其发生次数。该查询扫描4.2TB的压
缩数据。
```sql
Q5: SELECT TOP(aid, 20), COUNT(*) FROM T4
WHERE bid = {value1} AND cid = {value2}
```

使用系统的四种配置（范围从1000到4000个节点）执行查询。 执行时间在图13中。在每次运行中，总的CPU使用时间几乎相同，大约为300K秒，而用户感
知的时间随着系统规模的增大而几乎呈线性下降。 这一结果表明，较大的系统在资源使用方面可以与较小的系统一样有效，但可以更快地执行。

*落后者*。 我们的最后一个实验显示了落后者的影响。 下面的查询Q6在10的十二次方行的表T5上运行。 与其他数据集相比，T5是双副本的。 因此，落
后者减慢执行速度的可能性更高，因为reschedule job的机会更少。
```sql
Q6: SELECT COUNT(DISTINCT a) FROM T5
```
查询Q6读取超过1TB的压缩数据。 检索到的字段的压缩率约为10。如图14所示，每个插槽每个tablet中99％的tablet的处理时间低于5秒。 但是，当在
2500节点系统上执行时，一小部分tablet会花费更长的时间，从而将查询响应时间从不到一分钟拖至几分钟。 下一节总结了我们的实验结果和我们吸取的教训。

## 8. OBSERVATIONS(意见？诊断？)
Dremel每月扫描记录上千万亿。 图15显示了一个Dremel系统在典型的每月工作量中的查询响应时间分布，以对数规模。 如图所示，大多数查询都在10秒
以内完成，并且在交互式范围内。 一些查询在共享群集上实现了接近每秒1000亿条记录的扫描吞吐量，在专用计算机上甚至更高。 上面提供的实验数据表
明以下几点：
* 基于扫描的查询可以在高达一万亿条记录的磁盘驻留数据集上以交互速度执行。
* 对于包含数千个节点的系统，可以实现列和服务器数量的近线性可伸缩性。
* MR可以像DBMS一样受益于列式存储。
* record assembly和解析非常昂贵。 软件层（超出查询处理层）需要优化，以直接使用面向列的数据。
* MR和查询处理可以互补使用。 一层的输出可以提供另一层的输入。
* 在多用户环境中，较大的系统可以从规模经济中受益，同时可以提供质量上更好的用户体验。
* 如果可以接受以*准确性*为代价的速度权衡，则可以更早地终止查询，但仍可以覆盖大多数的数据。
* 网络规模数据集的大部分可以快速扫描。 在紧迫的时间范围内很难达到最后几个百分点。

Dremel的代码库很密集； 它包含少于10万行C ++，Java和Python代码。

## 9. 相关工作
MapReduce（MR）[12]框架旨在解决长期运行的批处理作业中的大规模计算难题。与MR一样，Dremel提供了容错执行，灵活的数据模型和原位数据处理功
能。 MR的成功促成了广泛的第三方实现（尤其是开源Hadoop [15]），以及由Aster，Cloudera，Greenplum和Vertica等供应商提供的将并行DBMS与
MR相结合的许多混合系统。 HadoopDB [3]是此混合类别中的研究系统。最近的文章[13，22]比较了MR和并行DBMS。我们的工作强调两种范式的互补性。

Dremel设计用于大规模运行。尽管可以将并行DBMS扩展到成千上万个节点，但是我们还没有发现任何尝试这样做的published work或行业报告。我们也不
曾熟知有关列式存储的MR研究的现有文献。

我们嵌套数据的列存储形式基于可追溯到几十年的思想：结构与内容的分离，以及转置的表示。最近对列存储工作的评论，包括压缩和查询处理可以在[1]中
找到。许多商业DBMS支持使用XML存储嵌套数据（例如[19]）。 XML存储方案试图将结构与内容分离，但由于XML数据模型的灵活性而面临更多挑战。使用
列式XML表示的一种系统是XMill [17]。 XMill是一种压缩工具。它存储所有组合字段的结构，并且不适合于选择性地检索列。 

Dremel中使用的数据模型是[2]中讨论的复杂值模型和嵌套关系模型的变体。 Dremel的查询语言基于[9]的思想，该思想引入了一种语言，该语言可避免
在访问嵌套数据时进行重组。相反，通常在XQuery和面向对象的查询语言中需要进行重组，例如，使用嵌套的for循环和构造函数。我们不知道[9]的实际
实现。 Pig [18]是最近一种对嵌套数据进行操作的类SQL语言。其他用于并行数据处理的系统包括Scope[6]和DryadLINQ [23]，并在[7]中进行了更详
细的讨论。

## 10. 结论
我们介绍了Dremel，这是一个用于大型数据集交互分析的分布式系统。 Dremel是基于简单组件构建的自定义，可扩展的数据管理解决方案。 它补充了MR
范式。 我们讨论了它在 trillion-record的数TB的真实数据集上的性能。 我们概述了Dremel的关键方面，包括其存储格式，查询语言和执行。 将来，
我们计划更深入地涵盖formal algebraic specification，join，可扩展性机制等领域。

## 11. 致谢

Dremel受益于Google的许多工程师和实习生的投入，特别是 Craig Chambers, Ori Gershoni, Rajeev Byrisetti, Leon Wong, Erik 
Hendriks, Erika Rice Scherpelz, Charlie Garrett, Idan Avraham, Rajesh Rao, Andy Kreling, Li Yin, Madhusudan 
Hosaagrahara, Dan Belov, Brian Bershad, Lawrence You, Rongrong Zhong, Meelap Shah, and Nathan Bales.

## 12. 引用
[1] D. J. Abadi, P. A. Boncz, and S. Harizopoulos.
Column-Oriented Database Systems. VLDB, 2(2), 2009.

[2] S. Abiteboul, R. Hull, and V. Vianu. Foundations of
Databases. Addison Wesley, 1995.

[3] A. Abouzeid, K. Bajda-Pawlikowski, D. J. Abadi, A. Rasin,
and A. Silberschatz. HadoopDB: An Architectural Hybrid of
MapReduce and DBMS Technologies for Analytical
Workloads. VLDB, 2(1), 2009.

[4] Z. Bar-Yossef, T. S. Jayram, R. Kumar, D. Sivakumar, and
L. Trevisan. Counting Distinct Elements in a Data Stream. In
RANDOM, pages 1–10, 2002.

[5] L. A. Barroso and U. H¨olzle. The Datacenter as a Computer:
An Introduction to the Design of Warehouse-Scale Machines.
Morgan & Claypool Publishers, 2009.

[6] R. Chaiken, B. Jenkins, P.-A. Larson, B. Ramsey, D. Shakib,
S. Weaver, and J. Zhou. SCOPE: Easy and Efficient Parallel
Processing of Massive Data Sets. VLDB, 1(2), 2008.

[7] C. Chambers, A. Raniwala, F. Perry, S. Adams, R. Henry,
R. Bradshaw, and N. Weizenbaum. FlumeJava: Easy,
Efficient Data-Parallel Pipelines. In PLDI, 2010.

[8] F. Chang, J. Dean, S. Ghemawat, W. C. Hsieh, D. A.
Wallach, M. Burrows, T. Chandra, A. Fikes, and R. Gruber.
Bigtable: A Distributed Storage System for Structured Data.
In OSDI, 2006.

[9] L. S. Colby. A Recursive Algebra and Query Optimization
for Nested Relations. SIGMOD Rec., 18(2), 1989.

[10] G. Czajkowski. Sorting 1PB with MapReduce. Official
Google Blog, Nov. 2008. At http://googleblog.blogspot.com/
2008/11/sorting-1pb-with-mapreduce.html.

[11] J. Dean. Challenges in Building Large-Scale Information
Retrieval Systems: Invited Talk. In WSDM, 2009.

[12] J. Dean and S. Ghemawat. MapReduce: Simplified Data
Processing on Large Clusters. In OSDI, 2004.

[13] J. Dean and S. Ghemawat. MapReduce: a Flexible Data
Processing Tool. Commun. ACM, 53(1), 2010.

[14] S. Ghemawat, H. Gobioff, and S.-T. Leung. The Google File
System. In SOSP, 2003.

[15] Hadoop Apache Project. http://hadoop.apache.org.

[16] Hive. http://wiki.apache.org/hadoop/Hive, 2009.

[17] H. Liefke and D. Suciu. XMill: An Efficient Compressor for
XML Data. In SIGMOD, 2000.

[18] C. Olston, B. Reed, U. Srivastava, R. Kumar, and
A. Tomkins. Pig Latin: a Not-so-Foreign Language for Data
Processing. In SIGMOD, 2008.

[19] P. E. O’Neil, E. J. O’Neil, S. Pal, I. Cseri, G. Schaller, and N. Westbury. ORDPATHs: Insert-Friendly XML Node 
Labels. In SIGMOD, 2004.

[20] R. Pike, S. Dorward, R. Griesemer, and S. Quinlan. Interpreting the Data: Parallel Analysis with Sawzall. 
Scientific Programming, 13(4), 2005.

[21] Protocol Buffers: Developer Guide. Available at http://code.google.com/apis/protocolbuffers/docs/overview.html.

[22] M. Stonebraker, D. Abadi, D. J. DeWitt, S. Madden, E. Paulson, A. Pavlo, and A. Rasin. MapReduce and Parallel
DBMSs: Friends or Foes? Commun. ACM, 53(1), 2010.

[23] Y. Yu, M. Isard, D. Fetterly, M. Budiu, U´ . Erlingsson, P. K. Gunda, and J. Currey. DryadLINQ: A System for 
General-Purpose Distributed Data-Parallel Computing Using a High-Level Language. In OSDI, 2008.
