---
title: sparkStreaming入门
subtitle: sparkStreaming基本概念
description: sparkStreaming基本概念
keywords: [spark,streaming,概念,原理]
author: liyz
date: 2018-11-14
tags: [spark,sparkStreaming]
category: [streaming]
---

## **概述**

Spark Streaming 是 Spark Core API 的扩展, 它支持弹性的, 高吞吐的, 容错的实时数据流的处理. 数据可以通过多种数据源获取, 例如 Kafka, Flume, Kinesis 以及 TCP sockets, 也可以通过例如 `map`, `reduce`, `join`, `window` 等的高级函数组成的复杂算法处理. 最终, 处理后的数据可以输出到文件系统, 数据库以及实时仪表盘中. 事实上, 你还可以在 data streams（数据流）上使用 机器学习 以及 图计算算法.

**运行原理**

![1542184745858](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/1542184745858.png)

1. sparkStreaming不断的从Kafka等数据源获取数据（连续的数据流），并将这些数据按照周期划分成为batch
2. 将每个batch的数据提交给SparkEngine来处理（每个batch的处理实际上还是批处理，只不过批量很小，几乎解决了实时处理）
3. 整个过程是持续的，即不断的接收数据并处理数据和输出结果

## **DStream**

1. DStream : Discretized Stream 离散流
2. 为了便于理解，Spark Straming提出了DStream对象，代表一个连续不断的输入流
3. DStream是一个持续的RDD序列，每个RDD代表一个计算周期（DStream里面有多个RDD）
4. 所有应用在DStream上的操作，都会被映射为对DStream内部的RDD上的操作
5. DStream本质上是一个以时间为键，RDD为值的哈希表，保存了按时间顺序产生的RDD，。Spark Streaming每次将新产生的RDD添加到哈希表中，而对于已经不再需要的RDD则会从这个哈希表中删除，所以DStream也可以简单地理解为以时间为键的RDD的动态序列，。设批处理时间间隔为1s，下图为4s内产生的DStream示意图。![streaming-dstream](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/streaming-dstream.png)

**初始化注意点**:

- 一旦一个 context 已经启动，将不会有新的数据流的计算可以被创建或者添加到它。.
- 一旦一个 context 已经停止，它不会被重新启动.
- 同一时间内在 JVM 中只有一个 StreamingContext 可以被激活.
- 在 StreamingContext 上的 stop() 同样也停止了 SparkContext 。为了只停止 StreamingContext ，设置 `stop()` 的可选参数，名叫 `stopSparkContext` 为 false.
- 一个 SparkContext 就可以被重用以创建多个 StreamingContexts，只要前一个 StreamingContext 在下一个StreamingContext 被创建之前停止（不停止 SparkContext）.

### **DStream输入源**

- Basic sources
  - file systems： `sparkContext.textFileStream(dir)`
    - 只监控指定文件夹中的文件，不监控里面的文件夹
    - 以文件的修改时间为准
    - 一旦开始处理，对文件的修改在当前窗口不会被读取
    - 文件夹下面文件越多扫描时间越长（和文件是否修改无关）
    - hdfs在打开输出流的时候就设置了更新时间，这个时候write操作还未完成就被读，可以先将文件写到一个未被监控的文件夹，待write 操作完成后，再移入监控的文件夹中
  - socket connections:`sparkContext.socketTextStream()`
  - Akka actors
- Advanced sources
  - Kafka: `KafkaUtils.createStream(ssc,zkQuorum,group,topicMap)`
  - Flume
  - Kinesis
  - Twitter 
- multiple input stream
  - ssc.union(Seq(stream1,stream2,...))
  - stream1.union(stream2)

**Batch duration**

对于源源不断的数据，Spark Streaming是通过切分的方式，先将连续的数据流进行离散化处理。数据流每被切分一次，对应生成一个RDD，每个RDD都包含了一个时间间隔内所获取到的所有数据。

批处理时间间隔的设置会伴随Spark Streaming应用程序的整个生命周期，无法在程序运行期间动态修改

1. duration设置：`new StreamingContext(sparkConf,Seconds(1))`
2. Spark Streaming按照设置的batch duration来积累数据，周期结束时把周期内的数据作为一个RDD，并添加任务给Spark Engine
3. batch duration的大小决定了Spark Streaming提交作业的频率和处理延迟
4. batch duration大小设定取决于用户需求，一般不会太大

### **Receiver接收器**

![1542187582639](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/1542187582639.png)

1. 除了FileInputDStream,其余输入源都会关联一个Receiver。
2. receiver以任务的形式运行在应用的执行器进程中，从输入源收集数据并保存为RDD。
3. receiver会将接收到的数据复制到另一个工作节点上进行加工处理。
4. core的最小数量是2，一个负责接收，一个负责处理（fileSysInput除外）
5. 分配给处理数据的cores应该多余分配给receivers的数量

**转换操作**

- 无状态操作
  - 和spark core语义一致
  - 对DStream的transform操作，实际作用于DStream中的每一个RDD
  - 如果DStream没有提供RDD操作，可通过transform函数实现,`dstream.transform(fun)`
  - 不能跨多个batch中的RDD执行
- 有状态操作
  - updateStateByKey  ：定一个一个状态和更新函数，返回新的状态,updateStateByKey必须配置检查点
  - window: 流式计算是周期性进行的，有时处理处理当前周期的数据，还需要处理最近几个周期的数据，这时候就需要窗口操作方法了。我们可以设置数据滑动窗口，将数个原始Dstream合并成一个窗口DStream。window操作默认执行persist in mermory
  - ![streaming-dstream-window](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/streaming-dstream-window.png)
    - windowDuration： 窗口时间间隔又称为窗口长度，它是一个抽象的时间概念，决定了Spark Streaming对RDD序列进行处理的范围与粒度，即用户可以通过设置窗口长度来对一定时间范围内的数据进行统计和分析
    - bathDuration: batch大小
    - 每次计算的batch数：`windowDuration/batchDuration`
    - slideDuration: 滑动时间间隔，控制多长时间计算一次默认和batchDuration相等

**操作规约**

普通规约是每次把window里面每个RDD都计算一遍，增量规约是每次只计算新进入window的数据，然后减去离开window的数据，得到的就是window数据的大小，在使用上，增量规约需要提供一个规约函数的逆函数，比如`+`对应的逆函数为`-`

- 普通规约：`val wordCounts=words.map(x=>(x,1)).reduceByKeyAndWindow(_+_,Seconds(5s),seconds(1))`

- 增量规约：`val wordCounts=words.map(x=>(x+1)).reduceByKeyAndWindow(_+_,_-_,Seconds(5s),seconds(1))`



### **DStream输出**

1. 输出操作：print,foreachRDD,saveAsObjectFiles,saveAsTextFiles,saveAsHadoopFiles
2. 碰到输出操作时开始计算求值
3. 输出操作特点：惰性求值
4. 最佳建立链接的方式

```scala
// 1. con't not create before foreachPartition function(cont't create in driver)
// 2. use foreachPartition instead of foreach
// 3. use connect pool instead of create connect every time
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

### **DSream持久化**

1. 默认持久化：`MEMORY_ONLY_SER`
2. 对于来源于网络的数据源（kafka,flume等）: `MEMORY_AND_DISK_SER_2`
3. 对于window操作默认进行`MEMORY_ONLY`持久化

## **checkpoint容错**

sparkStreaming 周期性的把应用数据存储到HDFS等可靠的存储系统中可以供回复时使用的机制叫做检查点机制，

作用：

1. 控制发生失败时需要计算的状态数：通过lineage重算，检查点机制可以控制需要在Lineage中回溯多远
2. 提供驱动器程序（driver）的容错:可以重新启动驱动程序，并让驱动程序从检查点恢复，这样spark streaming就可以读取之前运行的程序处理数据的进度，并从哪里开始继续。

数据类型：

- Metadata(元数据)： streaming计算逻辑，主要来恢复driver。

  - `Configuration`:配置文件,用于创建该streaming application的所有配置

  - `DStream operations`:对DStream进行转换的操作集合
  - `Incomplete batches`:未完成batchs，那些提交了job在队列等待尚未完成的job信息。
- `Data checkpointing`: 已经生成的RDD但还未保存到HDFS或者会影响后续RDD的生成。

注意点

1. 对于window和stateful操作必须指定checkpint
2. 默认按照batch duration来做checkpoint

**Checkpoint类**

checkpoint的形式是将类CheckPoint的实例序列化后写入外部内存

![1542198071413](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/1542198071413.png)

**缺点**

SparkStreaming 的checkpoint机制是对CheckPoint对象进行序列化后的数据进行存储，那么SparkStreaming Application重新编译后，再去反序列化checkpoint数据就会失败，这个时候必须新建StreamingContext

针对这种情况，在结合SparkStreaming+kafka的应用中，需要自行维护消费offsets，这样即使重新编译了application,还是可以从需要的offsets来消费数据。对于其他情况需要结合实际的需求进行处理。

**使用**

checkpoint的时间间隔正常情况下应该是sliding interval的5-10倍，可通过`dstream.checkpoint(checkpointInterval)`配置每个流的interval。

如果想要application能从driver失败中恢复，则application需要满足

- 若application首次重启，将创建一个新的StreamContext实例
- 若application从失败中重启，将会从chekcpoint目录导入chekpoint数据来重新创建StreamingContext实例

```
def createStreamingContext()={
    ...
    val sparkConf=new SparkConf().setAppName("xxx")
    val ssc=new StreamingContext(sparkConf,Seconds(1))
    ssc.checkpoint(checkpointDir)
}
...
val ssc=StreamingContext.getOrCreate(checkpointDir,createSreamingContext _)
```

**Accumulators, Broadcast Variables, and Checkpoints**

在sparkStreaming中累加器和广播变量不能够在checkpoints中恢复,广播变量是在driver上执行的，但是当driver重启后并没有执行广播，当slaves调用广播变量时报`Exception: (Exception("Broadcast variable '0' not loaded!",)`

可以为累加器和广播变量创建延迟实例化的单例实例，以便在驱动程序重新启动失败后重新实例化它们

问题参考：https://issues.apache.org/jira/browse/SPARK-5206



## **容错**

系统的容错主要从三个方面，接收数据，数据处理和输出数据，在sparkStreaming中，接收数据和数据来源有关系，处理数据可以保证exactly once,输出数据可以保证at least once。



### **输入容错**

sparStreaming并不能完全的像RDD那样实现lineage,因为其有的数据源是通过网络传输的，不能够重复获取。

接收数据根据数据源不同容错级别不同

- `with file`:通过hdfs等文件系统中读取数据时可以保证exactly-once
- `with reciever-base-source`:
  - `reliable reciever`:当reciever接收失败时不给数据源答复接收成功，在reciever重启后继续接收
  - `unreliable reciever`:接收数据后不给数据源返回接收结果，则数据源也不会再次下发数据

sparkStreaming通过write-ahead-logs 提供了at least once的保证。在spark1.3版本之后，针对kafka数据源，可以做到exactly once ,[更多内容](http://spark.apache.org/docs/latest/streaming-kafka-integration.html)

### **输出容错**

类似于foreachRdd操作，可以保证at least once，如果输出时想实现exactly once可通过以下两种方式：

- `Idempotent updates`:幂等更新，多次尝试将数据写入同一个文件
- `Transactional updates`:事物更新，实现方式：通过batch time和the index of rdd实现RDD的唯一标识，通过唯一标识去更新外部系统，即如果已经存在则跳过更新，如果不存在则更新。eg:


```scala
  dstream.foreachRDD { (rdd, time) =>
    rdd.foreachPartition { partitionIterator =>
      val partitionId = TaskContext.get.partitionId()
      val uniqueId = generateUniqueId(time.milliseconds, partitionId)
      // use this uniqueId to transactionally commit the data in partitionIterator
    }
  }
```

## **调优**

sparkStreaming调优主要从两方面进行：开源节流——提高处理速度和减少输入数据。

- 行时间优化
  - 设置合理的批处理时间和窗口大小
  - 提高并行度
    - 增加接收器数目
    - 将接收到数据重新分区
    - 提高聚合计算的并行度，例如对reduceByKey等shuffle操作设置跟高的并行度
- 内存使用与垃圾回收
  - 控制批处理时间间隔内的数据量
  - 及时清理不再使用的数据
  - 减少序列化和反序列化负担



详情参考：http://spark.apache.org/docs/latest/tuning.html#level-of-parallelism

原文：[streaming-programing-guide](ttps://spark.apache.org/docs/2.2.0/streaming-programming-guide.html)



