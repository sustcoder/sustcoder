---
title:spark and kafka
subtitle: spark和kafka的链接
description: spark和kafka问题
keywords: [kafka,spark]
date: 2019-02-02
tags: [kafka,spark]
category: [kafka]
---

##### spark从kafka读数据并发问题

默认情况下一个kafka分区对应一个RDD分区，但是有些场景下存在部分kafka分区处理较慢的场景，首先应该考虑是否是数据倾斜导致，以及增加kafka分区数量是否可以解决，如果解决不了可通过以下思路解决：

- 重写kafkaRDD的getPartition方法，让一个spark分区对应多个kafka分区，缺点是破坏了一个kafka分区内数据有序性问题。
- 在处理数据前重新分区，通过repartition或者coalease方法对数据重新分区。缺点是需要额外的时间和IO的浪费
- 在rdd.mapPartitions方法里面创建多个线程来处理同一个RDD分区里面的数据

[查看详情](https://www.iteblog.com/archives/2419.html)

##### 将RDBMS的数据实时传送到hive中

将binglog通过kafka发送到broker上，通过flume代理作为消费者消费数据并写入fluem

![kafka9](E:\data\oss\kafka\kafka9.png)

[查看详情](https://www.iteblog.com/archives/1771.html)

##### spark streaming kafka实现数据零丢失的几种方式

1. kafka高级API

使用Receiver接收数据，并开启WAL，将存储级别修改为`StorageLevel.MEMORY_AND_DISK_SER`。开启WAL后能够保证receiver在正常工作的情况下能够把接收到的消息存入WAL，但是如果receiver被强制中断的话依旧存在问题，因此可在streaming程序最后添加代码，确定wal执行完毕后再终止程序。通过receiver的方式只能实现at least once。

2. kafka Direct API

kafka direct api的方式是指自己维护offsets进行checkpoint操作，在receiver重启时重新从checkpoint开始消费，但是会存在checkpoint里保存的offset已经处理过的情况，需要手动去重新制定checkpoint。也可以存入zookeeper中。

```scala
messages.foreachRDD(rdd=>{
   val message = rdd.map(_._2)  
   //对数据进行一些操作
   message.map(method)
   //更新zk上的offset (自己实现)
   updateZKOffsets(rdd)
})
```



