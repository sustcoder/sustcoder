---
title: sparkStreaming源码解析之Job动态生成
subtitle: sparkStream的Job动态生成思维脑图
description: sparkStream的Job动态生成思维脑图
keywords: [spark,streaming,源码,JOB]
author: liyz
date: 2018-12-03
tags: [spark,源码解析]
category: [spark]
---

此文是从思维导图中导出稍作调整后生成的，思维脑图对代码浏览支持不是很好，为了更好阅读体验，文中涉及到的源码都是删除掉不必要的代码后的伪代码，如需获取更好阅读体验可下载脑图配合阅读：

 此博文共分为四个部分：

1. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsA82A.tmp.jpg)[DAG定义](https://sustcoder.github.io/2018/12/01/sparkStreaming-sourceCodeAnalysis_DAG/)
2. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wpsC3C6.tmp.jpg)[Job动态生成](https://sustcoder.github.io/2018/12/03/sparkStreaming-sourceCodeAnalysis_job/)
3. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps230.tmp.jpg)[数据的产生与导入](https://sustcoder.github.io/2018/12/09/sparkStreaming-sourceCodeAnalysis_dataInputOutput/)
4. ![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps614F.tmp.jpg)[容错](https://sustcoder.github.io/2018/12/12/sparkStreaming-sourceCodeAnalysis__faultTolerance/)

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps1C70.tmp.jpg)

在 Spark Streaming 程序的入口，我们都会定义一个 batchDuration，就是需要每隔多长时间就比照静态的 DStreamGraph 来动态生成一个 RDD DAG 实例。在 Spark Streaming 里，总体负责动态作业调度的具体类是 JobScheduler。

 JobScheduler 有两个非常重要的成员：JobGenerator 和 ReceiverTracker。JobScheduler 将每个 batch 的 RDD DAG 具体生成工作委托给 JobGenerator，而将源头输入数据的记录工作委托给 ReceiverTracker。

# 1. **启动**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps2045.tmp.jpg) 

## 1.1. **JobScheduler**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps2046.tmp.jpg) 

 

**job运行的总指挥是JobScheduler.start()，**

 JobScheduler 有两个非常重要的成员：JobGenerator 和 ReceiverTracker。JobScheduler 将每个 batch 的 RDD DAG 具体生成工作委托给 JobGenerator，而将源头输入数据的记录工作委托给 ReceiverTracker。

**在StreamingContext中启动scheduler**

```scala
class StreamingContext(sc,cp,batchDur){
	val scheduler = new JobScheduler(this)
	start(){
		scheduler.start()
	}
}
```

**在JobScheduler中启动recieverTracker和JobGenerator**

```scala
 class JobScheduler(ssc) {
	var receiverTracker:ReceiverTracker=null
	var jobGenerator=new JobGenerator(this)
	val jobExecutor=ThreadUtils.newDaemonFixedThreadPool()
	if(stared) return // 只启动一次
	receiverTracker.start()
    jobGenerator.start()
}
```

### 1.1.1. **启动ReceiverTracker**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps2057.tmp.jpg) 

1. 在JobScheduler的start中启动ReceiverTraker:`receiverTracker.start()： `

2. RecieverTracker 调用launchReceivers方法

```scala
class  ReceiverTracker {
	var endpoint:RpcEndpointRef=null
	def start()=synchronized{
		endpoint=ssc.env.rpcEnv.setEndpoint(
			"receiverTracker",
			new ReceiverTrackerEndpoint() 
		)
		launchReceivers()
	}
}
```

#### 1.1.1.1. **ReceiverSupervisor**

 ReceiverTracker将RDD DAG和启动receiver的Func包装成ReceiverSupervisor发送到最优的Excutor节点上

#### 1.1.1.2. **拉起receivers**

 从ReceiverInputDStreams中获取Receivers，并把他们发送到所有的worker nodes:

```
class  ReceiverTracker {
	var endpoint:RpcEndpointRef=
	private def launchReceivers(){
		// DStreamGraph的属性inputStreams
		val receivers=inputStreams.map{nis=>
			val rcvr=nis.getReceiver()
			// rcvr是对kafka,socket等接受数据的定义
			rcvr
		}	
		// 发送到worker
		 endpoint.send(StartAllReceivers(receivers))
	}
}
```

### 1.1.2. **启动DAG生成**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps2058.tmp.jpg) 

 在JobScheduler的start中启动JobGenerator:`JobGenerator.start()`

#### 1.1.2.1. **startFirstTime**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps2059.tmp.jpg) 

  **首次启动**

```scala
private def startFirstTime() {
	// 定义定时器
    val startTime = 
	new Time(timer.getStartTime())
	// 启动DStreamGraph
    graph.start(startTime - graph.batchDuration)
    //  启动定时器
	 timer.start(startTime.milliseconds)
}
```

##### 1.1.2.1.1. **启动DAG**

**graph的生成是在StreamingContext中**：

```scala
val graph: DStreamGraph={
	// 重启服务时
	if（isCheckpointPresent）{
		checkPoint.graph.setContext(this)
		checkPoint.graph.restoreCheckPointData()
		checkPoint.graph
	}else{
	// 首次初始化时
		val newGraph=new DStreamGraph()
		newGraph.setBatchDuration(_batchDur)
		newGraph
	}
}
```

**在GenerateJobs中启动graph**：

```scala
graph.start(nowTime-batchDuration)
```

##### 1.1.2.1.2. **启动timer**

**JobGenerator中定义了一个定时器：**

```scala
val timer=new RecurringTimer(colck,batchDuaraion,
		longTime=>eventLoop.post(
            GenerateJobs(
				new Time(longTime)
            )
         )
)
```

**在JobGenerator启动时会开始执行这个调度器：**

```scala
timer.start(startTime.milliseconds)
```

## 1.2. **RecurringTimer：定时器**

// 来自 JobGenerator

 ```scala
private[streaming]
class JobGenerator(jobScheduler: JobScheduler) extends Logging {
...
  private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
      longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
...
}
 ```

通过代码也可以看到，整个 timer 的调度周期就是 batchDuration，每次调度起来就是做一个非常简单的工作：往 eventLoop 里发送一个消息 —— 该为当前 batch (new Time(longTime)) GenerateJobs 了！

# 2. **生成**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps205A.tmp.jpg) 

**JobGenerator中定义了一个定时器，在定时器中启动生成job操作**

```scala
class JobGenerator:
// 定义定时器
val timer=
	new RecurringTimer(colck,batchDuaraion,
	longTime=>eventLoop.post(GenerateJobs(
	new Time(longTime))))
 
private def generateJobs(time: Time) {
  Try {
      
  // 1. 将已收到的数据进行一次 allocate
  receiverTracker.allocateBlocksToBatch(time)  
      
  //   2. 复制一份新的DAG实例
  graph.generateJobs(time)                                                 
   } match {
     case Success(jobs) =>
      
  // 3. 获取 meta 信息
  val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)  
      
  // 4. 提交job     
 jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))   
    case Failure(e) =>
      jobScheduler.reportError("Error generating jobs for time " + time, e)
  }
  // 5. checkpoint
  eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))      
}
```



## 2.2. **获取DAG实例** 

在生成Job并提交到excutor的第二步，

JobGenerator->DStreamGraph->OutputStreams->ForEachDStream->TransformationDStream->InputDStream

具体流程是：

\- 1. JobGenerator调用了DStreamGraph里面的gererateJobs(time)方法

\- 2. DStreamGraph里的generateJobs方法遍历了outputStreams

\- 3. OutputStreams调用了其generateJob(time)方法

\- 4. ForEachDStream实现了generateJob方法，调用了：

​	parent.getOrCompute(time)

递归的调用父类的getOrCompute方法去动态生成物理DAG图

# 3. **运行**

## 3.1. **异步处理:JobScheduler**

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps2081.tmp.jpg) 

**JobScheduler通过线程池执行从JobGenerator提交过来的Job，jobExecutor异步的去处理提交的job**

 ```scala
class JobScheduler{
  numConcurrentJobs = ssc.conf.getInt("spark.streaming.concurrentJobs", 1)
  val jobExecutor =ThreadUtils.
	newDaemonFixedThreadPool(numConcurrentJobs, "streaming-job-executor")

    def submitJobSet(jobSet: JobSet) {
		jobSet.jobs.foreach(job =>
                            jobExecutor.execute(new JobHandler(job)))
}
 ```

### 3.1.1. **Job:类比Thread**

### 3.1.2. **JobHandler：真正执行job**

 JobHandler 除了做一些状态记录外，最主要的就是调用 job.run()，

 在 ForEachDStream.generateJob(time) 时，是定义了 Job 的运行逻辑，即定义了 Job.func。而在 **JobHandler 这里，是真正调用了 Job.run()、将触发 Job.func 的真正执行**！

```scala
// 来自 JobHandler
def run()
{
  ...
  // 【发布 JobStarted 消息】
  _eventLoop.post(JobStarted(job))
  PairRDDFunctions.disableOutputSpecValidation.withValue(true) {
    // 【主要逻辑，直接调用了 job.run()】
    job.run()
  }
  _eventLoop = eventLoop
  if (_eventLoop != null) {
  // 【发布 JobCompleted 消息】
    _eventLoop.post(JobCompleted(job))
  }
  ...
}
```

### 3.1.3. **concurrentJobs : job并行度**

**spark.streaming.concurrentJobs job并行度**

这里 jobExecutor 的线程池大小，是由 spark.streaming.concurrentJobs 参数来控制的，当没有显式设置时，其取值为 1。

进一步说，这里 jobExecutor 的线程池大小，就是能够并行执行的 Job 数。而回想前文讲解的 DStreamGraph.generateJobs(time) 过程，一次 batch 产生一个 Seq[Job}，里面可能包含多个 Job —— 所以，确切的，有几个 output 操作，就调用几次 ForEachDStream.generatorJob(time)，就产生出几个 Job



**脑图制作参考**：https://github.com/lw-lin/CoolplaySpark

**完整脑图链接地址**：https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/spark-streaming-all.png