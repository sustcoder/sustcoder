---
title: sparkStreaming源码解析之DAG定义
subtitle: sparkStream的DAG定义源码解析
description: sparkStream的DAG定义源码解析
keywords: [spark,streaming,源码,DAG]
author: liyz
date: 2018-12-01
tags: [spark,源码解析]
category: [spark]
---

此文是从思维导图中导出稍作调整后生成的，思维脑图对代码浏览支持不是很好，为了更好阅读体验，文中涉及到的源码都是删除掉不必要的代码后的伪代码，如需获取更好阅读体验可下载脑图配合阅读：

 此博文共分为四个部分：

1. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA82A.tmp.jpg)[DAG定义](https://sustcoder.github.io/2018/12/01/sparkStreaming-sourceCodeAnalysis_DAG/)
2. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsC3C6.tmp.jpg)[Job动态生成](https://sustcoder.github.io/2018/12/03/sparkStreaming-sourceCodeAnalysis_job/)
3. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps230.tmp.jpg)[数据的产生与导入](https://sustcoder.github.io/2018/12/09/sparkStreaming-sourceCodeAnalysis_DataInputOutput/)
4. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps614F.tmp.jpg)[容错](https://sustcoder.github.io/2018/12/12/sparkStreaming-sourceCodeAnalysis_faultTolerance/)

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/DAG.jpg)



![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA82B.tmp.jpg) 

# 1. **DStream**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA82C.tmp.jpg) 

## 1.1. **RDD**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA83D.tmp.jpg) 

 

**DStream和RDD关系：**

```sca
DStream is a continuous sequence of RDDs：
generatedRDDs=new HashMap[Time,RDD[T]]()
```

### 1.1.1. **存储**

 **存储格式**

DStream内部通过一个HashMap的变量generatedRDD来记录生成的RDD:

```scal
 private[streaming] var generatedRDDs = new HashMap[Time, RDD[T]] ()
```

*其中 ：*

​	*- key: time是生成当前batch的时间戳*

​	*- value: 生成的RDD实例*

 **每一个不同的 DStream 实例，都有一个自己的 generatedRDD，即每个转换操作的结果都会保留**
### 1.1.2. **获取**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA83E.tmp.jpg) 

#### 1.1.2.1. **getOrCompute**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA83F.tmp.jpg) 

1. 从rdd的map中获取：generatedRDDs.get(time).orElse

2. map中没有则计算：val newRDD=compute(time)

3. 将计算的newRDD放入map中：generatedRDDs.put(time, newRDD)

 其中compute方法有以下特点：

- 不同DStream的计算方式不同

-  inputStream会对接对应数据源的API

-  transformStream会从父依赖中去获取RDD并进行转换得新的DStream

compute方法实现：

```scala
class ReceiverInputDStream{

 override def compute(validTime: Time): Option[RDD[T]] = {

    val blockRDD = {
 
      if (validTime < graph.startTime) {

        
        // If this is called for any time before the start time of the context,

        // then this returns an empty RDD. This may happen when recovering from a

        // driver failure without any write ahead log to recover pre-failure data.

        new BlockRDD[T](ssc.sc, Array.empty)

      } else {

        // Otherwise, ask the tracker for all the blocks that have been allocated to this stream

        // for this batch

        val receiverTracker = ssc.scheduler.receiverTracker

        val blockInfos = receiverTracker.getBlocksOfBatch(validTime).getOrElse(id, Seq.empty)

 
        // Register the input blocks information into InputInfoTracker

        val inputInfo = StreamInputInfo(id, blockInfos.flatMap(_.numRecords).sum)

        ssc.scheduler.inputInfoTracker.reportInfo(validTime, inputInfo)

 
        // Create the BlockRDD

        createBlockRDD(validTime, blockInfos)

      }

    }

    Some(blockRDD)

  }
```



### 1.1.3. **生成**

RDD主要分为以下三个过程：InputStream -> TransFormationStream -> OutputStream

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA864.tmp.jpg) 

#### 1.1.3.1. **InputStream**

inputstream包括FileInputStream，KafkaInputStream等等

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA865.tmp.jpg) 

##### 1.1.3.1.1. **FileInputStream**

FileInputStream的生成步骤：

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA866.tmp.jpg) 

1. 找到新产生的文件：val newFiles = findNewFiles(validTime.milliseconds)

2.  将newFiles转换为RDDs：val rdds=filesToRDD(newFiles)

   2.1.  遍历文件列表获取生成RDD: val fileRDDs=files.map(file=>newAPIHadoop(file))

   2.2.  将每个文件的RDD进行合并并返回：return new UnionRDD(fileRDDs)

3. 返回生成的rdds

#### 1.1.3.2. **TransformationStream**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA87D.tmp.jpg) 

RDD的转换实现：

1.  获取parent DStream：val parentDs=parent.getOrCompute(validTime)
2.  执行转换函数并返回转换结果：return parentDs.map(mapFunc)



**转换类的DStream实现特点**：

- 传入parent DStream和转换函数

- compute方法中从parent DStream中获取DStream并对其作用转换函数

 ```scala
private[streaming]

class MappedDStream[T: ClassTag, U: ClassTag] (
    parent: DStream[T],
    mapFunc: T => U
  ) extends DStream[U](parent.ssc) {

  override def dependencies: List[DStream[_]] = List(parent)
  override def slideDuration: Duration = parent.slideDuration
  override def compute(validTime: Time): Option[RDD[U]] = {
    parent.getOrCompute(validTime).map(_.map[U](mapFunc))
  }
}
 ```

不同DStream的getOrCompute方法实现：

- FilteredDStream：`parent.getOrCompute(validTime).map(_.filter(filterFunc)`
- FlatMapValuedDStream:`parent.getOrCompute(validTime).map(_.flatMapValues[U](flatMapValueFunc)`
- MappedDStream:`parent.getOrCompute(validTime).map(_.map[U](mapFunc))`

在最开始， **DStream 的 transformation 的 API 设计与 RDD 的 transformation 设计保持了一致，就使得，每一个 dStreamA.transformation() 得到的新 dStreamB 能将 dStreamA.transformation() 操作完美复制为每个 batch 的 rddA.transformation() 操作**。这也就是 DStream 能够作为 RDD 模板，在每个 batch 里实例化 RDD 的根本原因。



#### 1.1.3.3. **OutputDStream**

OutputDStream的操作最后都转换到ForEachDStream(),ForeachDStream中会生成Job并返回。

 **伪代码**

```scala
 def generateJob(time:Time){
	val jobFunc=()=>crateRDD{
		foreachFunc(rdd,time)
	}
	Some(new Job(time,jobFunc))
}
```

**源码**

```scala
private[streaming]
class ForEachDStream[T: ClassTag] (
    parent: DStream[T],
    foreachFunc: (RDD[T], Time) => Unit
  ) extends DStream[Unit](parent.ssc) {

  override def dependencies: List[DStream[_]] = List(parent)
  override def slideDuration: Duration = parent.slideDuration
  override def compute(validTime: Time): Option[RDD[Unit]] = None

  override def generateJob(time: Time): Option[Job] = {
    parent.getOrCompute(time) match {
      case Some(rdd) =>
        val jobFunc = () => createRDDWithLocalProperties(time) {
          ssc.sparkContext.setCallSite(creationSite)
          foreachFunc(rdd, time)
        }
        Some(new Job(time, jobFunc))
        case None => None
    }
  }
}
```

通过对output stream节点进行遍历，就可以得到所有上游依赖的DStream,直至找到没有父依赖的inputStream。

## 1.2. **特征**

 DStream基本属性:

- 父依赖： dependencies: List[DStream[_]]

- 时间间隔：slideDuration:Duration

- 生成RDD的函数：compute

## 1.3. **实现类**

DStream的实现类可分为三种：输入，转换和输出



![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA880.tmp.jpg) 

 

DStream之间的转换类似于RDD之间的转换，对于wordCount的例子，实现代码：

```scala
val lines=ssc.socketTextStream(ip,port)
val worlds=lines.flatMap(_.split("_"))
val pairs=words.map(word=>(word,1))
val wordCounts=pairs.reduceByKey(_+_)
wordCounts.print()
```

每个函数的返回对象用具体实现代替：

 ```scala
val lines=new SocketInputDStream(ip,port)
val words=new FlatMappedDStream(lines,_.split("_"))
val pairs=new MappedDStream(words,word=>(word,1))
val wordCounts=new ShuffledDStream(pairs,_+_)
new ForeachDStream(wordCounts,cnt=>cnt.print())
 ```

### 1.3.1. **ForeachDStream**

 DStream的实现分为两种，transformation和output

不同的转换操作有其对应的DStream实现，所有的output操作只对应于ForeachDStream

### 1.3.2. **Transformed DStream**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA890.tmp.jpg) 

### 1.3.3. **InputDStream**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA891.tmp.jpg) 

# 2. **DStreamGraph**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA892.tmp.jpg) 

##  2.1 **DAG分类**

- 逻辑DAG: 通过transformation操作正向生成

- 物理DAG: 惰性求值的原因，在遇到output操作时根据dependency逆向宽度优先遍历求值。

## 2.2 DAG生成

 **DStreamGraph属性**

```scala
 inputStreams=new ArrayBuffer[InputDStream[_]]()
 outputStreams=new ArrayBuffer[DStream[_]]()
```

**DAG实现过程**

​	通过对output stream节点进行遍历，就可以得到所有上游依赖的DStream,直至找到没有父依赖的inputStream。

​	sparkStreaming 记录整个DStream DAG的方式就是通过一个DStreamGraph 实例记录了到所有output stream节点的引用

**generateJobs**

```scala
def generateJobs(time: Time): Seq[Job] = {
      val jobs = this.synchronized {
      outputStreams.flatMap { 
		outputStream =>
        val jobOption = 
			// 调用了foreachDStream来生成每个job
			outputStream.generateJob(time)   jobOption.foreach(_.setCallSite(outputStream.creationSite))
        jobOption
      }
    }
	// 返回生成的Job列表
    jobs
  }
```





 **脑图制作参考**：https://github.com/lw-lin/CoolplaySpark

**完整脑图链接地址**：https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/spark-streaming-all.png