---
title: spark exception
subtitle: 异常处理记录
description: 常见异常解决方案
keywords: [spark,倾斜,调优]
date: 2019-01-25
tags: [spark,调优,转载]
category: [spark]
---



**异常信息**

运行开发的应用程序时，**当Driver内存不足时，Spark任务会挂起，不能退出应用程序**。

```scala
Exception in thread "handle-read-write-executor-1" java.lang.OutOfMemoryError: Java heap space
    at java.nio.HeapByteBuffer.<init>(HeapByteBuffer.java:57)
    at java.nio.ByteBuffer.allocate(ByteBuffer.java:331)
    at org.apache.spark.network.nio.Message$.create(Message.scala:90)
    at org.apache.spark.network.nio.ReceivingConnection$Inbox.org$apache$spark$network$nio$ReceivingConnection$Inbox$$createNewMessage$1(Connection.scala:454)
    at org.apache.spark.network.nio.ReceivingConnection$Inbox$$anonfun$1.apply(Connection.scala:464)
    at org.apache.spark.network.nio.ReceivingConnection$Inbox$$anonfun$1.apply(Connection.scala:464)
    at scala.collection.mutable.MapLike$class.getOrElseUpdate(MapLike.scala:189)
    at scala.collection.mutable.AbstractMap.getOrElseUpdate(Map.scala:91)
    at org.apache.spark.network.nio.ReceivingConnection$Inbox.getChunk(Connection.scala:464)
    at org.apache.spark.network.nio.ReceivingConnection.read(Connection.scala:541)
    at org.apache.spark.network.nio.ConnectionManager$$anon$6.run(ConnectionManager.scala:198)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)
```

**可能原因**

因为提供给应用程序的Driver内存不足，导致内存空间出现full gc（全量垃圾回收）。但Garbage Collectors未能及时清理内存空间，使得full gc状态一直存在，导致了应用程序不能退出。

定位思路

- 执行命令强制将任务退出，然后通过修改内存参数的方式解决内存不足的问题，使任务执行成功。
- 针对此类数据量大的任务，希望任务不再挂起，遇到内存不足时，直接提示任务运行失败。

处理步骤

- 首先执行如下步骤退出此应用程序。

  - 进入Spark的Driver端删除任务进程。说明：

    根据实际提交应用程序的进程号，请谨慎操作，避免误删其他应用程序。

    执行如下命令查询应用程序的进程号：`ps -ef | grep SparkSubmit`

    执行如下命令删除对应的进程号：`kill <进程号> `

  - 执行Yarn命令删除任务进程。

    `yarn application -kill <App ID>`

- 您可以执行如下操作步骤解决内存不足的问题，以便任务能够执行成功。

  在提交任务执行命令时配置内存参数，将内存参数配置为更大的值，以满足任务需求。

```scala
bin/spark-submit --master yarn-cluster 
--class com.spark.test.wordcountWithSave 
--driver-memory 5g 
--executor-memory 4g 
/opt/client/Spark/spark/lib/spark-test.jar 
hdfs://hacluster/data.txt collect 
```
  其中Driver端和Executor端得内存在如下参数中设置：

  - Driver端：--driver-memory

  - Executor端：--executor-memory

- （可选）当任务需要的资源很大，且此时存在内存不足的问题，您希望任务直接执行失败，而不是一直挂起，无法判断任务是否执行成功。您可以通过设置以下参数以满足以上机制。如果您不需要设置为此机制，跳过此步骤。

  在客户端的安装目录下，修改“$spark_client/Spark/spark/conf/spark-defaults.conf”文件，添加“spark.yarn.appMasterEnv.SPARK_USE_CONC_INCR_GC”参数并配置该参数的值为“true”。

  例如：

```scala
cd /opt/spark_client/Spark/spark/conf
vi spark-defaults.conf
// 添加以下内容：设置SPARK_USE_CONC_INCR_GC为true的话，就使用CMS的垃圾回收机制
// cms回收机制：GC过程短暂停，适合对时延要求较高的服务，用户线程不允许长时间的停顿。
spark.yarn.appMasterEnv.SPARK_USE_CONC_INCR_GC=true
```

重新提交应用程序，当出现内存不足时，应用程序不再挂起，会出现任务执行失败的提示。

**对应源码**

```scala
// 如果一个机器上有多个container,一个container的GC会影响到其他container,使用CMS的内存管理。    
// TODO: Remove once cpuset version is pushed out.
 // The context is, default gc for server class machines ends up using all cores to do gc -
// hence if there are multiple containers in same node, Spark GC affects all other containers'
// performance (which can be that of other Spark containers)
// Instead of using this, rely on cpusets by YARN to enforce "proper" Spark behavior in
// multi-tenant environments. Not sure how default Java GC behaves if it is limited to subset
// of cores on a node.
    val useConcurrentAndIncrementalGC = launchEnv.get("SPARK_USE_CONC_INCR_GC").exists(_.toBoolean)
    if (useConcurrentAndIncrementalGC) {
      // In our expts, using (default) throughput collector has severe perf ramifications in
      // multi-tenant machines
      javaOpts += "-XX:+UseConcMarkSweepGC"
      javaOpts += "-XX:MaxTenuringThreshold=31"
      javaOpts += "-XX:SurvivorRatio=8"
      javaOpts += "-XX:+CMSIncrementalMode"
      javaOpts += "-XX:+CMSIncrementalPacing"
      javaOpts += "-XX:CMSIncrementalDutyCycleMin=0"
      javaOpts += "-XX:CMSIncrementalDutyCycle=10"
    }
```



**问题现象**

 探索环境中的yarn的ResourceManager的内存随着使用时间不断的变大，最终导致用户无法访问yarn的资源管理页面，并且整个集群的调度变得异常缓慢，最终导致隔一段时间就需要重启一下Yarn。

**问题原因**

主要原因是由于消费者（也就是往TimelineServer发送Event事件数据）发送数据的能力不够，导致消息队列中缓存的大量的消息，随着时间的推移，消息数据不断增多，从而导致yarn内存不够用，从而需要重启。

**解决办法**

- 将`yarn.system-metrics-publisher.enabled`的值设置为false，也就是关闭了发送测量数据的功能。

- 通过分析SystemMetricsPublisher的源码发现，在创建分发器的时候，有个参数`yarn. system-metrics-publisher.dispatcher.pool-size `，默认值为10）可以控制发送的线程池的数量，通过增大该值，理论上应该也是可以解决该问题的，但没有具体测试，因为是线上环境。

[查看详情](https://www.jianshu.com/p/9d5b052a2bc0)



**问题现象**

程序跑5分钟小数据量日志不到5分钟，但相同的程序跑一天大数据量日志各种失败。经优化，使用160 vcores + 480G memory，一天的日志可在2.5小时内跑完

**问题原因**

随着数据量的增大和程序运行时间的增长，由于各种原因导致的性能问题拙见凸显，最终导致job执行失败。

**解决办法**

优化方向有三个方面：基础算子优化减少shuffle,合理分配partition，利用缓存等，资源优化例如给每个executor的内存大小,yarn的资源分配等，最后再从JVM的GC机制上优化。

**对于大数据量缓存级别设置为DISK_ONLY**

对于5分钟小数据量，采用`StorageLevel.MEMORY_ONLY`，而对于大数据下我们直接采用了`StorageLevel.DISK_ONLY`。DISK_ONLY_2相较DISK_ONLY具有2备份，cache的稳定性更高，但同时开销更大，cache除了在executor本地进行存储外，还需走网络传输至其他节点。后续我们的优化，会保证executor的稳定性，故没有必要采用DISK_ONLY_2。实时上，如果优化的不好，我们发现executor也会大面积挂掉，这时候即便DISK_ONLY_2，也是然并卵，所以保证executor的稳定性才是保证cache稳定性的关键。

错误使用cache和对lazy理解错误的场景：

此时还没执行算子,cache操作不生效，会导致重复的获取file数据，可以在baseRDD.cache()后增加baseRDD.count()，显式的触发cache，当然count()是一个action，本身会触发一个job

```scala
val raw = sc.textFile(file)
val baseRDD = raw.map(...).filter(...)
baseRDD.cache() 
val threadList = new Array(
  new Thread(new SubTaskThead1(baseRDD)),
  new Thread(new SubTaskThead2(baseRDD)),
  new Thread(new SubTaskThead3(baseRDD))
)
threadList.map(_.start())
threadList.map(_.join())
```

由于textFile()也是lazy执行的，故本例会进行两次相同的hdfs文件的读取，效率较差。解决办法，是对pvLog和clLog共同的父RDD进行cache。

```scala
val raw = sc.textFile(file)
val pvLog = raw.filter(isPV(_))
val clLog = raw.filter(isCL(_))
val baseRDD = pvLog.union(clLog)
val baseRDD.count()
```



MemoryOverhead是JVM进程中除Java堆以外占用的空间大小，包括方法区（永久代）、Java虚拟机栈、本地方法栈、JVM进程本身所用的内存、直接内存（Direct Memory）等。通过`spark.yarn.executor.memoryOverhead`设置，单位MB。



如果用于**存储RDD的空间不足，先存储的RDD的分区会被后存储的覆盖**。当需要使用丢失分区的数据时，丢失的数据会被重新计算。

如果**Java堆或者永久代的内存不足，则会产生各种OOM异常，executor会被结束**。spark会重新申请一个container运行executor。失败executor上的任务和存储的数据会在其他executor上重新计算。

**如果实际运行过程中ExecutorMemory+MemoryOverhead之和（JVM进程总内存）超过container的容量。YARN会直接杀死container。executor日志中不会有异常记录**。spark同样会重新申请container运行executor。

在Java堆以外的JVM进程内存占用较多的情况下，应该将MemoryOverhead设置为一个足够大的值，应该将MemoryOverhead设置为一个足够大的值，以防JVM进程因实际占用的内存超标而被kill。如果默认值（math.max((MEMORY_OVERHEAD_FACTOR *executorMemory).toInt,MEMORY_OVERHEAD_MIN）不够大，可以通过spark.yarn.executor.memoryOverhead手动设置一个更大的值。



**问题现象**

spark-submit 提交的spark任务已经结束, 但是spark-submit没有结束。

**问题原因**

job在运行完毕后，需要将数据保存到hdfs上，在保存到hdfs方法里对数据进行了两次复制，如果结果数据量较大则导致spark job已经全部完成了，但是我们的程序还执行着。

**解决办法**

通过配置写数据时的参数，减少一次数据的复制操作。

- 直接在 `conf/spark-defaults.conf` 里面设置 `spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version 2`，这个是全局影响的。
- 直接在 Spark 程序里面设置，`spark.conf.set("mapreduce.fileoutputcommitter.algorithm.version", "2")`，这个是作业级别的。
- 如果你是使用 Dataset API 写数据到 HDFS，那么你可以这么设置 `dataset.write.option("mapreduce.fileoutputcommitter.algorithm.version", "2")`。

[查看详情](http://hbase.group/question/331)

## 数据本地化

参数：`spark.locality.wait`默认为3s,

原因：增加等待时间可最大可能的拿到本地数据。

数据本地化级别：

- 进程本地化（process_local）：task要计算的数据在同一个executor中
- 节点本地化（node_local）:
  - task要计算的数据在同一个Worker的不同Executor进程中
  - task要计算的数据在同一个Worker磁盘上,或在HDFS上，恰好有block同一个节点上
- 机架本地化（rack_local）:
  - 在executor的磁盘
  - 在executor内存

Spark中的数据位置获取：

在提交任务时会确定每个task的最优位置。

DAGScheduler切割Job，划分Stage, 通过调用submitStage来提交一个Stage对应的tasks，submitStage会调用submitMissingTasks,submitMissingTasks 确定每个需要计算的 task 的preferredLocations，通过调用getPreferrdeLocations()得到partition 的优先位置，就是这个 partition 对应的 task 的优先位置，对于要提交到TaskScheduler的TaskSet中的每一个task，该task优先位置与其对应的partition对应的优先位置一致。

Spark中数据本地化流程：

TaskScheduler在发送task的时候，会根据数据所在的节点发送task,这时候的数据本地化的级别是最高的，如果这个task在这个Executor中等待了三秒，重试发射了5次还是依然无法执行，那么TaskScheduler就会认为这个Executor的计算资源满了，TaskScheduler会降低一级数据本地化的级别，重新发送task到其他的Executor中执行，如果还是依然无法执行，那么继续降低数据本地化的级别...

  **如果想让每一个task都能拿到最好的数据本地化级别，那么调优点就是等待时间加长。注意！如果过度调大等待时间，虽然为每一个task都拿到了最好的数据本地化级别，但是我们job执行的时间也会随之延长**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/522d5951fa4a69047c5130d4bc9260cd256.jpg)

[查看更多...](https://my.oschina.net/u/2000675/blog/2999579)

## Spark写Redis实践总结

Redis作为机器学习算法与后台服务器的媒介，算法计算用户数据并写入Redis；后台服务器读取Redis，并为前端提供实时接口

```scala
 // 写Redis
sampleData.repartition(500).foreachPartition(rows => {
  val rc = new Jedis(redisHost, redisPort)
  rc.auth(redisPassword)
  val pipe = rc.pipelined 
  rows.foreach(r => {
    val redisKey = r.getAs[String]("key")
    val redisValue = r.getAs[String]("value")
    pipe.set(redisKey, redisValue)
    pipe.expire(redisKey, expireDays * 3600 * 24)
  })
  pipe.sync()
})
```

1. 使用pipe批量插入提高插入速度

2. 使用foreachPartition减少客户端链接占用

3. 使用repartition(500)防止大partition导致内存溢出

   [查看详情..](http://slamke.github.io/2017/10/30/Spark%E5%86%99Redis%E5%AE%9E%E8%B7%B5%E6%80%BB%E7%BB%93/)



点：

1. 定制序列化方法,减少序列化后的存储占用`spark.serializer=org.apache.spark.serializer.KryoSerializer`
2. 分区过少
   - 重新分区，**重新分区会发生一次shuffle操作**
   - 在输入接口将数据分成更多的片
   - 在写hdfs时,设置更小的block
3. 参数`spark.default.parallelism`,此参数值默认偏小，可适当调大，比如hdfs默认和一个block块一样的，导致task任务较少，设置的executor数量等参数不起作用
4. 

## 合理的设置executor数量和内存

![log1](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/log1.png)



![log2](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/log2.png)

## 

































