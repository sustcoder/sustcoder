---
title: sparkSql jion优化
subtitle: spakSql jion详解
description: jion优化解读
keywords: [sparkSql,jion,解析]
author: liyz
date: 2019-01-11
tags: [sparkSql,jion]
category: [spark]
---

## jion分类

当前SparkSQL支持三种Join算法－shuffle hash join、broadcast hash join以及sort merge join。其中前两者归根到底都属于hash join，只不过在hash join之前需要先shuffle还是先broadcast。

选择思路大概是：大表与小表进行join会使用broadcast hash join，一旦小表稍微大点不再适合广播分发就会选择shuffle hash join，最后，两张大表的话无疑选择sort merge join。

### **Hash Jion**

将小表转换成Hash Table,用大表进行遍历，对每个元素去Hash Table里查看是否存在，存在则进行jion。

先来看看这样一条SQL语句：`select * from order,item where item.id = order.i_id`，很简单一个Join节点，参与join的两张表是item和order，join key分别是item.id以及order.i_id。现在假设这个Join采用的是hash join算法，整个过程会经历三步：

1. 确定Build Table以及Probe Table：这个概念比较重要，Build Table使用join key构建Hash Table，而Probe Table使用join key进行探测，探测成功就可以join在一起。通常情况下，小表会作为Build Table，大表作为Probe Table。此事例中item为Build Table，order为Probe Table。

2. 构建Hash Table：依次读取Build Table（item）的数据，对于每一行数据根据join key（item.id）进行hash，hash到对应的Bucket，生成hash table中的一条记录。数据缓存在内存中，如果内存放不下需要dump到外存。

3. 探测(Probe)：再依次扫描Probe Table（order）的数据，使用相同的hash函数映射Hash Table中的记录，映射成功之后再检查join条件（item.id = order.i_id），如果匹配成功就可以将两者join在一起。

基本流程可以参考上图，这里有两个小问题需要关注：

1. hash join性能如何？很显然，hash join基本都只扫描两表一次，可以认为o(a+b)

2. 为什么Build Table选择小表？道理很简单，因为构建的Hash Table最好能全部加载在内存，效率最高；这也决定了hash join算法只适合至少一个小表的join场景，对于两个大表的join场景并不适用；

上文说过，hash join是传统数据库中的单机join算法，在分布式环境下需要经过一定的分布式改造，说到底就是尽可能利用分布式计算资源进行并行化计算，提高总体效率。hash join分布式改造一般有两种经典方案：

1. broadcast hash join：将其中一张小表广播分发到另一张大表所在的分区节点上，分别并发地与其上的分区记录进行hash join。broadcast适用于小表很小，可以直接广播的场景。

2. shuffler hash join：一旦小表数据量较大，此时就不再适合进行广播分发。这种情况下，可以根据join key相同必然分区相同的原理，将两张表分别按照join key进行重新组织分区，这样就可以将join分而治之，划分为很多小join，充分利用集群资源并行化。

### **Broadcast Hash Join**

broadcast hash join可以分为两步：

1. broadcast阶段：将小表广播分发到大表所在的所有主机。广播算法可以有很多，最简单的是先发给driver，driver再统一分发给所有executor；要不就是基于bittorrete的p2p思路；

2. hash join阶段：在每个executor上执行单机版hash join，小表映射，大表试探；

SparkSQL规定broadcast hash join执行的基本条件为被广播小表必须小于参数`spark.sql.autoBroadcastJoinThreshold`，默认为10M。

**BroadcastNestedLoopJoin**

`cross jion`在进行笛卡尔集运算时使用了嵌套云环的jion方式,也就是我们常用的两个for循环嵌套进行判断。

### **Shuffle Hash Join**

在大数据条件下如果一张表很小，执行join操作最优的选择无疑是broadcast hash join，效率最高。但是一旦小表数据量增大，广播所需内存、带宽等资源必然就会太大，broadcast hash join就不再是最优方案。此时可以按照join key进行分区，根据key相同必然分区相同的原理，就可以将大表join分而治之，划分为很多小表的join，充分利用集群资源并行化。如下图所示，shuffle hash join也可以分为两步：

1. shuffle阶段：分别将两个表按照join key进行分区，将相同join key的记录重分布到同一节点，两张表的数据会被重分布到集群中所有节点。这个过程称为shuffle

2. hash join阶段：每个分区节点上的数据单独执行单机hash join算法。

### **Sort-Merge Join**

1. shuffle阶段：将两张大表根据join key进行重新分区，两张表数据会分布到整个集群，以便分布式并行处理

2. sort阶段：对单个分区节点的两表数据，分别进行排序

3. merge阶段：对排好序的两张分区表数据执行join操作。join操作很简单，分别遍历两个有序序列，碰到相同join key就merge输出（**相比hash jion解决了大表不能全部hash到内存中的问题**）

仔细分析的话会发现，sort-merge join的代价并不比shuffle hash join小，反而是多了很多。那为什么SparkSQL还会在两张大表的场景下选择使用sort-merge join算法呢？这和Spark的shuffle实现有关，**目前spark的shuffle实现都适用sort-based shuffle算法，因此在经过shuffle之后partition数据都是按照key排序的。因此理论上可以认为数据经过shuffle之后是不需要sort的，可以直接merge**。

## Hash Jion优化

hash  jion中使用了谓词[布隆过滤器](https://my.oschina.net/freelili/blog/3001045)进行下推实现了`FR(Runtime Filter)`，  对jion操作进一步做了优化。

### 谓词下推

- 其一是逻辑执行计划优化层面的说法，比如SQL语句：select * from order ,item where item.id = order.item_id and item.category = ‘book’，正常情况语法解析之后应该是先执行Join操作，再执行Filter操作。通过谓词下推，可以将Filter操作下推到Join操作之前执行。即将where item.category = ‘book’下推到 item.id = order.item_id之前先行执行。
- 其二是真正实现层面的说法，谓词下推是将过滤条件从计算进程下推到存储进程先行执行（存储与计算分离的场景）减少IO，网络，内存等开销。例如将filter(bloomfilter)操作从excutor(计算进程)中转移到datanode(存储进程)中完成。

## 存在问题

 由于spark [CBO](http://hbasefly.com/2017/05/04/bigdata%EF%BC%8Dcbo/)分析的不准确问题导致broadcastjoin 经常出现乱广播的情形。

## 参考

http://hbasefly.com/2017/03/19/sparksql-basic-join/

http://hbasefly.com/2017/04/10/bigdata-join-2/