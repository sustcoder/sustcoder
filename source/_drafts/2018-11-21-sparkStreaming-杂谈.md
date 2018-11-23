---
title: sparkStreaming杂谈
subtitle: sparkStreaming常见问题解析
description: sparkStreaming原理剖析
keywords: [spark,streaming,概念,原理]
author: liyz
date: 2018-11-21
tags: [spark,sparkStreaming]
category: [streaming]
---

- DStream与RDD区别与联系
- 怎么做到不重及exactly-once呢
- 一个inputStream对应一个Receiver对吧,一个Receiver会分配到一个Executor上,那么这个Receiver接收到的数据都会放在这个Executor上吧,这样会不会造成数据倾斜呢?
- 这多个batch可以并行处理吗，还是等一个batch执行完毕才会执行下一个？
- 一个batch里面有多个job，里面每个job完成都会进行一次checkpoint吗？ 还是说一个batch的全部job完成了才进行checkpoint
- 逻辑DAG与物理DAG
- kafka offset丢失
- kafka direct方式和Receiver-based读取数据
- Hbase checkAndPut保证原子性：如何加batch-id 
- 修改app 反序列化导致checkpoint失效
- spark.streaming.concurrentJobs大于1时会丢数据的问题
- 如果一个子RDD由两个父RDD生成，那么在迭代过程中是怎样完成的
-  Spark Streaming 的 JobSet, Job，与 Spark Core 的 Job, Stage, TaskSet, Task 这几个概念
- meta信息是timer信息吗？ batch时间取值怎么取的
- batch 时间来向 `ReceiverTracker` 查询得到划分到本 batch 的块数据 meta 信息

