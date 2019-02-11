---
title: sparkSql catalyst优化器
subtitle: catalyst优化器详解
description: catalyst优化器解析
keywords: [sparkSql,catalyst,解析]
author: liyz
date: 2019-01-10
tags: [sparkSql,优化]
category: [spark]
---

## 相关概念

- AST树
   SQL语法树是编译后被解析的树状结构，树包括很多对象，每个节点都有特定的数据类型，同事有孩子节点（TreeNode对象）。

- 规则
   等价规则转化将规则用于语法树。任何一个SQL优化器中，都会定义大量的Rule，SQL优化器遍历所有节点。匹配所有给定规则，如果匹配成功进行相应转换；失败则继续遍历下一个节点。

## **Catalyst工作流程**

- Parser，利用[ANTLR](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2Fantlr%2Fantlr4)将sparkSql字符串解析为抽象语法树AST，称为unresolved logical plan/ULP
- Analyzer，借助于数据元数据catalog将ULP解析为logical plan/LP
- Optimizer，根据各种RBO，CBO优化策略得到optimized logical plan/OLP，主要是对Logical Plan进行剪枝，合并等操作，进而删除掉一些无用计算，或对一些计算的多个步骤进行合并

## **优化分类**

**基于规则优化**/Rule Based Optimizer/RBO

- 一种经验式、启发式优化思路，基于已知的优化方法定义具体的规则进行优化。
- 对于核心优化算子join有点力不从心，如两张表执行join，到底使用broadcaseHashJoin还是sortMergeJoin，目前sparkSql是通过手工设定参数来确定的，如果一个表的数据量小于某个阈值（默认10M）就使用broadcastHashJoin

常见的三种规则优化：谓词下推、常量累加、列剪枝

- 谓词下推：扫描数据量过滤，比如合并两个过滤条件为一个减少一次过滤扫描
- 常量累加：减少常量操作,eg:从`100+80`优化为`180`避免每一条record都需要执行一次`100+80`的操作
- 列剪枝(去掉不需要的列)：对列式数据库提高扫描效率，减少网络、内存数据量消耗

**基于代价优化**/Cost Based Optimizer/CBO

许多基于规则的优化技术已经实现，但优化器本身仍然有很大的改进空间。例如，没有关于数据分布的详细列统计信息，因此难以精确地估计过滤（filter）、连接（join）等数据库操作符的输出大小和基数 (cardinality)。由于不准确的估计，它经常导致优化器产生次优的查询执行计划，此时就需要基于代价(io和cpu的消耗)的优化器，通过对表数据量等进行优化。

Spark SQL会依据逻辑执行计划生成至少一个物理执行计划，随后通过Cost Model对每个物理执行计划进行开销评估，并选择预估开销最小的一个作为最终的物理执行计划送去做代码生成。

具体步骤如下。

1. 首先采集原始表基本信息，例如：表数据量大小，行数，列信息（Max,min,非空,最大列长等）,列直方图

2. 再定义每种算子的基数评估规则，即一个数据集经过此算子执行之后基本信息变化规则。这两步完成之后就可以推导出整个执行计划树上所有中间结果集的数据基本信息

3. 定义每种算子的执行代价，结合中间结果集的基本信息，此时可以得出任意节点的执行代价

4. 将给定执行路径上所有算子的代价累加得到整棵语法树的代价

5. 计算出所有可能语法树代价，并选出一条代价最小的



**jion操作优化**：

1. 配置自动广播的阈值。`spark.sql.autoBroadcastJoinThreshold`默认值`10M`,`-1`代表不进行广播

2. 使用Executor广播减少Driver内存压力。默认的BroadCastJoin会将小表的内容，全部收集到Driver中，因此需要适当的调大Driver的内存，当广播任务比较频繁的时候，Driver有可能因为OOM而异常退出。

   此时，可以开启Executor广播，配置Executor广播参数`spark.sql.bigdata.useExecutorBroadcast`为`true`，减少Driver内存压力。

3. 小表执行超时，会导致任务结束。默认情况下，BroadCastJoin只允许小表计算5分钟，超过5分钟该任务会出现超时异常，而这个时候小表的broadcast任务依然在执行，造成资源浪费。

   这种情况下，有两种方式处理：

   - 调整`spark.sql.broadcastTimeout`的数值，加大超时的时间限制。
   - 降低`spark.sql.autoBroadcastJoinThreshold`的数值，不使用BroadCastJoin的优化。

4. spark内部优化。通过谓词下推和布隆过滤器对jion操作进行优化。

   - 针对每个join评估当前两张表使用每种join策略的代价，根据代价估算确定一种代价最小的方案

   - 不同physical plans输入到[代价模型](https://link.jianshu.com/?t=https%3A%2F%2Fwww.slideshare.net%2Fdatabricks%2Fcostbased-optimizer-in-apache-spark-22)（目前是统计），调整join顺序，减少中间shuffle数据集大小，达到最优输出

   - [详见...](https://my.oschina.net/freelili/blog/3001124)



**扩展数据源优化**

扩展数据源是指mySql,Hdfs等数据源，sparkSql针对外部数据源，对查询逻辑进行优化，使得尽可能少的数据被扫描到spark内存中。

例如：`SELECT * FROM MYSQL_TABLE WHERE id > 100`在没有优化前，会将表里面的所有数据先加载到spark内存中，然后进行筛选。在经过扩展数据源优化后，where 后面的过滤条件会在mysql中执行。



**执行计划查看**

```scala
// 物理逻辑计划，包括parsed logical plan,Analyzed Logical plan,Optimized logical plan
spark.sql("select * from tab0 ").queryExecution
// 查看物理执行计划
spark.sql("select * from tab0 ").explain
// 使用Spark WebUI进行查看
```



举例：

```scala
// 定义DF
scala> val df = park.read.option("header","true").csv("file:///user/iteblog/sales.csv")
df: org.apache.spark.sql.DataFrame = [transactionId: string, customerId: string ... 2 more fields]
// 给 amountPaid 列乘以1
scala> val multipliedDF = df.selectExpr("amountPaid * 1")
multipliedDF: org.apache.spark.sql.DataFrame = [(amountPaid * 1): double]
// 查看优化计划 
scala> println(multipliedDF.queryExecution.optimizedPlan.numberedTreeString)
00 Project [(cast(amountPaid#89 as double) * 1.0) AS (amountPaid * 1)#91]
01 +- Relation[transactionId#86,customerId#87,itemId#88,amountPaid#89] csv
```

Spark中的所有计划都是使用tree代表的。所以`numberedTreeString`方法以树形的方式打印出优化计划。正如上面的输出一样。

　　所有的计划都是从下往上读的。下面是树中的两个节点：
　　1、01 Relation：表示从csv文件创建的DataFrame；
　　2、00 Project：表示投影（也就是需要查询的列）。

从上面的输出可以看到，为了得到正确的结果，Spark通过`cast`将`amountPaid`转换成`double`类型。

## 自定义优化器

继承`RuleExecutor`类，实现其`apply`方法。不需要像UDF一样注册，默认碰到此场景会执行此优化规则。

Catalyst是一个单独的模块类库，这个模块是基于规则的系统。这个框架中的每个规则都是针对某个特定的情况来优化的。比如：`ConstantFolding`规则用于移除查询中的常量表达式。

Spark 2.0提供了API，我们可以基于这些API添加自定义的优化规则。实现可插拔方式的Catalyst规则添加。

```scala
object MultiplyOptimizationRule extends RuleExecutor[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = plan transformAllExpressions {
       case Multiply(left,right) if right.isInstanceOf[Literal] &&
           right.asInstanceOf[Literal].value.asInstanceOf[Double] == 1.0 =>
           println("optimization of one applied")
           left
       }
}
```



参考：

CBO详细规则：https://issues.apache.org/jira/secure/attachment/12823839/Spark_CBO_Design_Spec.pdf

华为给spark2.2.0提供了： [A cost-based optimizer framework for Spark SQL](https://issues.apache.org/jira/browse/SPARK-16026)

在Spark SQL中定义查询优化规则:https://www.iteblog.com/archives/1706.html#Catalyst

SparkSQL  Catalyst简介:http://hbasefly.com/2017/03/01/sparksql-catalyst/