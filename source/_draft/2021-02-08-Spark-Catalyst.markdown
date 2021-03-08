---
title:  "Spark Catalyst"
date:   2021-02-02 00:00:00
tags:
    - Spark
---


# Spark Catalyst

## Catalyst(催化剂)

* Spark SQL的核心, 可扩展优化器, 基于Scala的函数式编程，灵活可扩展
* 利用进阶的编程语言特性： (e.g. Scala’s pattern matching and quasiquotes) 
* 内部包含代表树的通用库，并且通过规则来操作树。

1. 有专门用来做relational query处理的库。
2. 有几套规则可以处理查询的不同阶段：analysis, logical optimization, physical planning, code generation -> 将查询编译为java字节码。
3. 使用scala的quasiquote，使运行时更容易生成code。
4. Catalyst 提供了几种public extension points, 包括外部数据源和用户自定义类型。
* 同时支持RBO和CBO

## Tree

Catalyst的主要数据类型是由node对象组成的tree。每个node包含node type 和 >=0的children。新的node type 被定义为 TreeNode的子类。这些对象immutable，并使用functional transformation来操作。

```
Literal(value: Int): 常量
Attribute(name: String): 输入行的某个属性， e.g.,"x"
Add(left: TreeNode, right: TreeNode): 两个表达式的求和
```

```
x + (1 + 2)
Add(Attribute(x), Add(Literal(1), Literal(2)))
```

![tree](/images/spark-catalyst/tree.png)


## Rules
Tree可以被Rules操作，Rules是将tree转换为另一个tree的函数。

最常见的方法是使用一组模式匹配，使用特定的结构查找和替换subtree。

优化：Catalyst，树提供了转换方法，应用模式匹配函数递归遍历树上的所有node。
示例：折叠常量之间的Add操作：
```
tree.transform {

  case Add(Literal(c1), Literal(c2)) => Literal(c1+c2)

}


一个transform调用可以匹配多个模式
tree.transform {

  case Add(Literal(c1), Literal(c2)) => Literal(c1+c2)

  case Add(left, Literal(0)) => left

  case Add(Literal(0), right) => right

}
```
实践过程中，rules需要被执行多次，已完整transform tree。Catalyst将rules组成batches， 一批一批地执行直到tree达到固定状态，也就是再次应用rules之后，tree不再发生变化。


## Spark SQL中Catalyst的应用
Catalyst 的tree transformation 框架共四步：
1. 分析logical plan，解决引用问题
2. logical plan optimization
3. physical planning
4. code generation: 将查询部分编译为java字节码
在physical plan阶段，Catalyst可能产生多个plan，根据CBO做比较，所有其他阶段都是纯RBO的；
每个阶段使用不同type 的tree nodes；
Catalyst包含了表达式节点，数据类型节点，逻辑操作符节点和物理操作符节点的库。
![phases](/images/spark-catalyst/phases.png)

### 1. 分析
Spark SQL输入是要被计算的关系：
1. SQL Parser产生的AST
2. 由API创建的DataFrame Object

关系中可能会包含未被解析的属性引用(unresolved attribute references)或者关系：

SELECT col FROM sales

col是否valid, col的类型在查找sales之前都未知，我们将类型未知或者尚未匹配到输入表的属性称为unresolved

Spark SQL使用Catalyst rules and a Catalog object来解析这些属性。创建一个unresolved logical plan树，并运用rules来操作
* 根据catalog中的name查询关系
* 映射命名属性
* 决定多个属性是否是相同引用，相同则分配相同的唯一ID
* 传播类型和强制类型转换：我们无法知道 1 + col 的类型，直到解析出col的类型，可能需要将子表达式转为兼容类型
如 1 + col： col为NUMBER，则1 + col为数值类型; col为string, 则1 + col为String
In total, the rules for the analyzer are about 1000 lines of code.
 

## Logical Optimizations
逻辑优化阶段应用标准RBO转为logical plan。CBO是先应用Rules生成多个plan，然后计算这些plan的costs。
Rules 包括：
* 常量折叠
* 谓词下推
* 映射裁剪
* null传播
* Boolean表达式简化...
In total, the logical optimization rules are 800 lines of code.

## Physical Planning
输入为logical plan，使用spark执行引擎的physical operators，生成一个或者多个physical plan，然后使用cost model选择一个plan。
At the moment, cost-based optimization is only used to select join algorithms: for relations that are known to be small, Spark SQL uses a broadcast join, using a peer-to-peer broadcast facility available in Spark. The framework supports broader use of cost-based optimization, however, as costs can be estimated recursively for a whole tree using a rule. We thus intend to implement richer cost-based optimization in the future.

The physical planner also performs rule-based physical optimizations, such as pipelining projections or filters into one Spark map operation. In addition, it can push operations from the logical plan into data sources that support predicate or projection pushdown. We will describe the API for these data sources in a later section.

In total, the physical planning rules are about 500 lines of code.


## Code Generation
Catalyst依赖使用Scala语言特性 quasiquotes ，简化code generation。
Quasiquotes 允许在Scala中进行AST的程序构造。

使用Catalyst 将表示表达式的tree转化为scala代码的AST，用于评估表达式，然后编译和运行生成的代码。

As a simple example, consider the Add, Attribute and Literal tree nodes introduced in Section 4.2, which allowed us to write expressions such as (x+y)+1. Without code generation, such expressions would have to be interpreted for each row of data, by walking down a tree of Add, Attribute and Literal nodes. This introduces large amounts of branches and virtual function calls that slow down execution. With code generation, we can write a function to translate a specific expression tree to a Scala AST as follows:

```
def compile(node: Node): AST = node match {

  case Literal(value) => q"$value"

  case Attribute(name) => q"row.get($name)"

  case Add(left, right) => q"${compile(left)} + ${compile(right)}"

}
```

In total, Catalyst’s code generator is about 700 lines of code.







[1] https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html
[2] Spark SQL: Relational Data Processing in Spark http://people.csail.mit.edu/matei/papers/2015/sigmod_spark_sql.pdf


所以，函数调用，CPU，指令，寄存器，缓存和内存之间的关系是怎样的？
unsafe 方法和机器指令
指令流水线

* 编译执行
* data-centric 