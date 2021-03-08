---
title:  "Spark Tungsten：优化Spark核心执行引擎"
date:   2021-02-07 00:00:00
tags:
    - Spark
---


# Spark Tungsten(钨)

## 1. 什么是Tungsten

Tungsten是自Spark项目创建以来对Spark执行引擎的最大修改，从本质上改善了Spark应用的内存和CPU执行效率，将性能向到现代硬件的极限推进。

事实上，Spark工作负载越来越受到CPU和内存的限制，而不是磁盘IO和网络通信，引发了大家对CPU执行效率的关注，最近的大数据工作负载性能研究表明了这一个趋势。

### 1.1 为什么CPU成为了瓶颈
* 硬件角度：现在的硬件配置提供了越来越大的总IO带宽，比如10Gbps的网络连接, 高带宽的SSD或者条带HDD阵列用于存储。
* 软件角度：spark optimizer现在通过修剪不必要的输入数据，可以避免大量磁盘IO的工作。在spark shuffle子系统中，序列化和hashing(CPU bound) 成了关键瓶颈，而不是底层硬件的网络吞吐量。


### 1.2 Tungsten包含了哪些优化
* 内存管理和二进制处理：利用应用程序语义(sun.misc.Unsafe)显式管理内存，消除JVM对象模型和GC的开销
* Cache-aware computation(缓存感知计算)： 利用memory hierarchy（内存阶层）的算法和数据结构
* Code Generation: 利用现代编译器和CPU生成代码
* 没有Virtual Function Dispatches： 减少复杂的CPU调用，虚函数调度多时，会对性能有影响
* 中间数据在内存 VS. 中间数据在CPU寄存器：Tungsten Phase 2将中间数据放到CPU寄存器，相比于从内存获取数据，从CPU寄存器获取数据的周期数减少了一个数量级。
* loop unrolling(循环展开)and SIMD: 更好地利用现代编译器和CPU能力进行编译，执行简单的循环，而不是复杂的函数调用。


## 2. 内存管理和二进制处理（binary processing）

JVM应用依赖垃圾收集器来管理内存，随着spark应用性能推到极限，JVM对象开销不可忽视。
### 2.1 Java对象占据了太多内存
JVM对象占据了大量的固有（inherent）内存开销。比如说，字符串"abcd"使用UTF-8编码时占用4个bytes。JVM本地的字符串实现，以不同方式存储以适应更多场景。每个character编码
使用UTF-16耗费2bytes， 每个String对象包含12bytes的header和8bytes的hashcode。
```
java.lang.String object internals:

OFFSET  SIZE   TYPE DESCRIPTION                    VALUE
     0     4        (object header)                ...
     4     4        (object header)                ...
     8     4        (object header)                ...
    12     4 char[] String.value                   []
    16     4    int String.hash                    0
    20     4    int String.hash32                  0
Instance size: 24 bytes (reported by Instrumentation API)
```
**一个简单的4bytes字符串在JVM对象模型中占用了超过48bytes！！！**

### 2.2 Java垃圾回收效率可能会很差

JVM对象模型的另一个重要开销是内存回收。年轻代需要高频地分配和回收内存，当GC可以可靠地估算对象的生命周期时会很有效，但是当GC不能有效估算对象的生命周期时，效果会很差。
因此如果想要获得性能，需要使用GC调优的黑魔法，使用大量关于对象生命周期的JVM参数，提供更多关于JVM的信息。

因为Spark应用不仅仅是通用程序，Spark了解数据如何在计算的各个阶段以及job和task的范围内流动。Spark比JVM垃圾收集器更了解内存块的生命周期，
可以比JVM更有效地管理内存。

### 2.3 通过显式管理内存和二进制数据（而不是java对象模型）来解决
为了解决对象开销和GC效率低下的问题，Spark引入了显式的内存管理器，将大部分的Spark操作转换为直接操作二进制数据，而不是java对象。
这是基于JVM提供的```sun.misc.Unsafe```实现的高级功能，暴露了c-style的内存访问（显示分配，释放，指针计算）。
另外，Unsafe方法是固有(intrinsic)的，意味着每个方法调用都由JIT编译为单独的一条机器指令（machine instruction）。

Spark 1.4 引入了新的hash map对DataFrames和SQL聚合，1.5会有更多的数据结构来满足大部分操作，包括sorting和joins。

![new-hash-map](/images/tungsten/new-hash-map.png)
总的来说，通过显式管理内存的手段，直接操作二进制数据, 而不是java对象, 可以消除GC开销, 提升Spark性能。

## 3. Cache-aware Computation

### 3.1 什么是in-memory Computation

Spark可以高效利用集群内存资源，比基于磁盘处理的解决方案高效很多。Spark也可以处理远大于内存的数据量，将数据透明地spill到磁盘，执行外部操作，比如排序和hash。

### 3.2 什么是cache-aware Computation

相似地，cache-aware computation 通过对L1/L2/L3级别的CPU cache的高效利用，可以更快地处理数据，比利用主存(main memory)还要快好几个数量级。

在对Spark应用进行性能分析时，发现大部分的CPU时间都是在等待从main memory中fetch数据。因此设计了cache-friendly的算法和数据结构使得Spark
应用在提取数据上花费更少的时间。


### 3.3 举个栗子
记录（records）排序：标准排序过程是保存一个records的指针**数组**，使用快排来交换指针，直到所有records有序。因为顺序扫描访问，
所以排序很容易有好的cache hit rate，但是对指针**列表**排序，会有更低的cache hit rate，因为对比操作需要对两个(指向内存中任意位置的记录的)指针取消引用。
![cache-aware](/images/tungsten/cache-aware.png)

如何改善排序的缓存本地性？简单的方式是和指针一起，保存每个记录的key。如果sort key是64-bit integer, 就是用128-bit
(64-bit pointer和64-bit key)将每个记录保存在指针数组中。这样快排就可以使用pinter-key pairs，线性排序，而不需要随机内存访问。

在Spark中，大部分操作都是可以简化为a small list of operations，比如aggregations，sorting和join。通过改善这些operation的效率。
可以整体改善Spark应用的效率。

## 4. 代码生成
Spark 之前在SQL和DataFrames中引入了code generation for expression evaluation,  解析表达式（Expression Evaluation）是对于特定
记录如（```age = 37 ```）计算一个表达式如（``` age < 40 and age > 35 ```）的过程。

Spark在运行时动态生成字节码，用于解析这些表达式。而不是通过较慢的解释器逐步解析表达式。

相比于解释执行，代码生成：
1 减少了基础类型的装箱
2 更重要的是，避免昂贵的多态函数调度(虚函数调用)

### 4.1 vectorized expression
当前Spark已经把代码生成扩展应用到了大部分内置表达式，并且计划把代码生成的级别从record-at-a-time提升到vectorized expression, 通过JIT
的能力，利用现代CPU中更好的指令流水线，实现一次处理多条记录。

### 4.2 除了expression evaluation的代码生成, 还有其他领域的代码生成应用
1 内存二进制格式到wire-protocol的转换，用于shuffle。shuffle 当前经常以数据序列化为瓶颈，而不是下层的网络。通过代码生成，可以增加序列化
的吞吐量，相应的增加网路吞吐量。
![code-gen](/images/tungsten/codegen.png)
单线程，八百万复杂数据记录的性能 -> (Kryo vs. CodeGen) CodeGen的性能是Kryo性能的两倍


## 5 Tungsten和Tungsten之外

### 5.1 Spark应用Tungsten

Tungsten项目是Spark核心引擎设计的巨大变动。

Spark 1.4 在DataFrame API中实现了aggregation operations的内存显式管理，同时支持自定义序列化器。

Spark 1.5 扩展了二进制内存管理和cache-aware数据结构的覆盖范围。

### 5.2 Tungsten之外的待研究和优化点

研究对LLVM或者OpenCL的编译，因此Spark应用程序可以利用现代CPU中的SSE/SIMD指令，以及GPU中广泛的并行性来加快机器学习和图形计算的操作。


## 参考引用

[1] https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html

[2] https://databricks.com/glossary/tungsten

[3] https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html

[4] https://spoddutur.github.io/spark-notes/second_generation_tungsten_engine.html

[5] https://www.linkedin.com/pulse/catalyst-tungsten-apache-sparks-speeding-engine-deepak-rajak/?articleId=6674601890514378752

