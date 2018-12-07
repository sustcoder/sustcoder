---
title: sparkStreaming源码解析之容错
subtitle: sparkStream的数据容错机制
description: sparkStream的数据容错思维脑图
keywords: [spark,streaming,源码,容错]
author: liyz
date: 2018-12-12
tags: [spark,源码解析]
category: [spark]
---

此文是从思维导图中导出稍作调整后生成的，思维脑图对代码浏览支持不是很好，为了更好阅读体验，文中涉及到的源码都是删除掉不必要的代码后的伪代码，如需获取更好阅读体验可下载脑图配合阅读：

 此博文共分为四个部分：

1. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA82A.tmp.jpg)[DAG定义](https://sustcoder.github.io/2018/12/01/sparkStreaming-sourceCodeAnalysis_DAG/)
2. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsC3C6.tmp.jpg)[Job动态生成](https://sustcoder.github.io/2018/12/03/sparkStreaming-sourceCodeAnalysis_job/)
3. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps230.tmp.jpg)[数据的产生与导入](https://sustcoder.github.io/2018/12/09/sparkStreaming-sourceCodeAnalysis_dataInputOutput/)
4. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps614F.tmp.jpg)[容错](https://sustcoder.github.io/2018/12/12/sparkStreaming-sourceCodeAnalysis_faultTolerance/)

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps753E.tmp.jpg)

​	策略		优点			缺点

(1) 热备		无 recover time		需要占用双倍资源

(2) 冷备		十分可靠				存在 recover time

(3) 重放		不占用额外资源		存在 recover time

(4) 忽略		无 recover time		准确性有损失

# 1. **driver端容错**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9D6.tmp.jpg) 



# 2. **executor端容错**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9D7.tmp.jpg) 

## 2.1. **热备**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9D8.tmp.jpg) 

 

Receiver 收到的数据，通过 ReceiverSupervisorImpl，将数据交给 BlockManager 存储；而 BlockManager 本身支持将数据 replicate() 到另外的 executor 上，这样就完成了 Receiver 源头数据的热备过程。

 

而在计算时，计算任务首先将获取需要的块数据，这时如果一个 executor 失效导致一份数据丢失，那么计算任务将转而向另一个 executor 上的同一份数据获取数据。因为另一份块数据是现成的、不需要像冷备那样重新读取的，所以这里不会有 recovery time。

### 2.1.1. **备份**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9D9.tmp.jpg) 

备份流程：

​	先保存此block块，如果保存失败则不再进行备份，如果保存成功则获取保存的block块，执行复制操作。

```scala
class BlockManager {

	def doPutIterator(){

		doPut(blockId,level,tellMaster){

			// 存储数据

			if(level){

				memoryStore.putIteratorAsBytes（）

			}else if(level.useDisk){

				diskStore.put()

			}

			// 当前block已经存储成功则继续：		

			if(blockWasSuccessfullyStored){

				// 报告结果给master

				if(tellMaster){

					reportBlockStatus(blockid,status)

				}

				// 备份

				if(level.replication>1){

					// 从上面保存成功的位置获取block

					 val bytesToReplicate =doGetLocalBytes(blockId, info)

					// 正式备份

					replicate(

						blockId, 

						bytesToReplicate, 

						level

					)

				}

			}

		}

	}

}
```



### 2.1.2. **恢复**

 

计算任务首先将获取需要的块数据，这时如果一个 executor 失效导致一份数据丢失，那么计算任务将转而向另一个 executor 上的同一份数据获取数据。因为另一份块数据是现成的、不需要像冷备那样重新读取的，所以这里不会有 recovery time。



## 2.2. **冷备**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9DA.tmp.jpg) 

 

冷备是每次存储块数据时，除了存储到本 executor，还会把块数据作为 log 写出到 WriteAheadLog 里作为冷备。这样当 executor 失效时，就由另外的 executor 去读 WAL，再重做 log 来恢复块数据。WAL 通常写到可靠存储如 HDFS 上，所以恢复时可能需要一段 recover time

### 2.2.1. **WriteAheadLog**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9DB.tmp.jpg) 

 

WriteAheadLog 的特点是顺序写入，所以在做数据备份时效率较高，但在需要恢复数据时又需要顺序读取，所以需要一定 recovery time。

 

不过对于 Spark Streaming 的块数据冷备来讲，在恢复时也非常方便。这是因为，对某个块数据的操作只有一次（即新增块数据），而没有后续对块数据的追加、修改、删除操作，这就使得在 WAL 里只会有一条此块数据的 log entry。所以，我们在恢复时只要 seek 到这条 log entry 并读取就可以了，而不需要顺序读取整个 WAL。

 

**也就是，Spark Streaming 基于 WAL 冷备进行恢复，需要的 recovery time 只是 seek 到并读一条 log entry 的时间，而不是读取整个 WAL 的时间**，这个是个非常大的节省

 

#### 2.2.1.1. **配置**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9DC.tmp.jpg) 

##### 2.2.1.1.1. **存放目录配置**

 

WAL 存放的目录：`{checkpointDir}/receivedData/{receiverId}`

{checkpointDir} ：在 `ssc.checkpoint(checkpointDir) `指定的​	

{receiverId} ：是 Receiver 的 id

文件名：不同的 rolling log 文件的命名规则是 	`log-{startTime}-{stopTime}`



##### 2.2.1.1.2. **rolling配置**

 

FileBasedWriteAheadLog 的实现把 log 写到一个文件里（一般是 HDFS 等可靠存储上的文件），然后每隔一段时间就关闭已有文件，产生一些新文件继续写，也就是 rolling 写的方式



rolling 写的好处是单个文件不会太大，而且删除不用的旧数据特别方便



这里 rolling 的间隔是由参数 **spark.streaming.receiver.writeAheadLog.rollingIntervalSecs**（默认 = 60 秒） 控制的

 

#### 2.2.1.2. **读写对象管理**

 

WAL将读写对象和读写实现分离，由FileBasedWriterAheadLog管理读写对象，LogWriter和LogReader根据不同输出源实现其读写操作

 

class FileBasedWriteAheadLog:

 

write(byteBuffer:ByteBuffer,time:Long):

​	1.  先调用getCurrentWriter(),获取当前currentWriter.

​	2. 如果log file 需要rolling成新的，则currentWriter也需要更新为新的currentWriter

​	3. 调用writer.write(byteBuffer)进行写操作

​	4. 保存成功后返回： 

​		path:保存路径

​		offset:偏移量

​		length:长度



read(segment:WriteAheadRecordHandle):

​	ByteBuffer {}:

​	1. 直接调用reader.read(fileSegment)



read实现：

// 来自 FileBasedWriteAheadLogRandomReader

 

```scala
def read(

	segment: FileBasedWriteAheadLogSegment): ByteBuffer = synchronized {

  assertOpen()

  	// 【seek 到这条 log 所在的 offset】

 	 instream.seek(segment.offset)

 	 // 【读一下 length】

 	 val nextLength = instream.readInt()

  	 val buffer = new Array[Byte](nextLength)

  	 // 【读一下具体的内容】

  	 instream.readFully(buffer)

    // 【以 ByteBuffer 的形式，返回具体的内容】

    ByteBuffer.wrap(buffer)

}
```

 

## 2.3. **重放**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9DD.tmp.jpg) 

 

如果上游支持重放，比如 Apache Kafka，那么就可以选择不用热备或者冷备来另外存储数据了，而是在失效时换一个 executor 进行数据重放即可。

 

### 2.3.1. **基于Receiver**

 

**偏移量又kafka负责，有可能导致重复消费**

 

这种是将 Kafka Consumer 的偏移管理交给 Kafka —— 将存在 ZooKeeper 里，失效后由 Kafka 去基于 offset 进行重放

这样可能的问题是，Kafka 将同一个 offset 的数据，重放给两个 batch 实例 —— 从而只能保证 at least once 的语义

### 2.3.2. **Direct方式**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9DE.tmp.jpg) 

 

**偏移量由spark自己管理，可以保证exactly-once**

 

由 Spark Streaming 直接管理 offset —— 可以给定 offset 范围，直接去 Kafka 的硬盘上读数据，使用 Spark Streaming 自身的均衡来代替 Kafka 做的均衡

这样可以保证，每个 offset 范围属于且只属于一个 batch，从而保证 exactly-once

 

所以看 Direct 的方式，**归根结底是由 Spark Streaming 框架来负责整个 offset 的侦测、batch 分配、实际读取数据**；并且这些分 batch 的信息都是 checkpoint 到可靠存储（一般是 HDFS）了。这就没有用到 Kafka 使用 ZooKeeper 来均衡 consumer 和记录 offset 的功能，而是把 Kafka 直接当成一个底层的文件系统来使用了。

#### 2.3.2.1. **DirectKafkaInputDStream**

 

负责侦测最新 offset，并将 offset 分配至唯一个 batch

#### 2.3.2.2. **KafkaRDD**



负责去读指定 offset 范围内的数据，并基于此数据进行计算

## 2.4. **忽略**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsB9EF.tmp.jpg) 

### 2.4.1. **粗粒度忽略**

 

在driver端捕获job抛出的异常，防止当前job失败，这样做会忽略掉整个batch里面的数据

### 2.4.2. **细粒度忽略**

 

细粒度忽略是在excutor端进行的，如果接收的block失效后，将失败的Block忽略掉，只发送没有问题的block块到driver



**脑图制作参考**：https://github.com/lw-lin/CoolplaySpark

**完整脑图链接地址**：https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/spark-streaming-all.png