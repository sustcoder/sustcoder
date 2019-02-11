---
title: sparkCore源码解析之Job
subtitle: job生成过程解析
description: rdd的DAG到stage到task
keywords: [spark,core,源码,job]
author: liyz
date: 2019-01-14
tags: [spark,源码解析]
category: [spark]
---

 ![job](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/Job.png)

# 1. **概念**

站在不同的角度看job

1. transaction: Job是由一组RDD上转换和动作组成。

2. stage: Job是由ResultStage和多个ShuffleMapState组成

3. init:由action操作触发提交执行的一个函数
    action操作会触发调用sc.runJob方法，

Job是一组rdd的转换以及最后动作的操作集合，它是Spark里面计算最大最虚的概念，甚至在spark的任务页面中都无法看到job这个单位。 但是不管怎么样，在spark用户的角度，job是我们计算目标的单位，每次在一个rdd上做一个动作操作时，都会触发一个job，完成计算并返回我们想要的数据。
**Job是由一组RDD上转换和动作组成**，这组RDD之间的转换关系表现为一个有向无环图(DAG)，每个RDD的生成依赖于前面1个或多个RDD。
在Spark中，两个RDD之间的依赖关系是Spark的核心。站在RDD的角度，两者依赖表现为点对点依赖， 但是在Spark中，RDD存在分区（partition）的概念，两个RDD之间的转换会被细化为两个RDD分区之间的转换。
Stage的划分是对一个Job里面一系列RDD转换和动作进行划分。
首先job是因动作而产生，因此每个job肯定都有一个ResultStage，否则job就不会启动。
其次，如果Job内部RDD之间存在宽依赖，Spark会针对它产生一个中间Stage，即为ShuffleStage，严格来说应该是ShuffleMapStage，这个stage是针对父RDD而产生的， 相当于在父RDD上做一个父rdd.map().collect()的操作。ShuffleMapStage生成的map输入，对于子RDD，如果检测到所自己所“宽依赖”的stage完成计算，就可以启动一个shuffleFectch， 从而将父RDD输出的数据拉取过程，进行后续的计算。
  因此**一个Job由一个ResultStage和多个ShuffleMapStage组成**。

# 2. **job处理流程**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsA9AB.tmp.jpg) 
<https://github.com/ColZer/DigAndBuried/blob/master/spark/shuffle-study.md>

## 2.1. **job生成过程**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsA9AC.tmp.jpg) 
### 2.1.1. **job重载函数**
调用SparkContext里面的函数重载，将分区数量，需要计算的分区下标等参数设置好
以rdd.count为例：
```scala
rdd.count
// 获取分区数
sc.runJob(this, Utils.getIteratorSize _).sum
// 设置需要计算的分区
runJob(rdd, func, 0 until rdd.partitions.length)
// 设置需要在每个partition上执行的函数
runJob(rdd, (ctx: TaskContext, it: Iterator[T]) => cleanedFunc(it), partitions)
```
### 2.1.2. **设置回调函数**
定义一个接收计算结果的对象数组并将其返回
构造一个Array,并构造一个函数对象"(index, res) => results(index) = res"继续传递给runJob函数,然后等待runJob函数运行结束,将results返回; 对这里的解释相当在runJob添加一个回调函数,将runJob的运行结果保存到Array到, 回调函数,index表示mapindex, res为单个map的运行结果
```scala
  def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int]): Array[U] = {
	// 定义返回的结果集
    val results = new Array[U](partitions.size)
	// 定义resulthandler
    runJob[T, U](rdd, func, partitions, (index, res) => results(index) = res)
	// 返回计算结果
    results
  }
```
### 2.1.3. **获取需要执行的excutor**
将需要执行excutor的地址和回调函数等传给DAG调度器，由DAG调度器进行具体的submitJob操作。
```scala
def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
	// 获取需要发送的excutor地址
    val callSite = getCallSite
	// 闭包封装，防止序列化错误
    val cleanedFunc = clean(func)
	// 提交给dag调度器,
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    // docheckpoint
    rdd.doCheckpoint()
  }
```
注意：**dagScheduler.runJob是堵塞的操作,即直到Spark完成Job的运行之前,rdd.doCheckpoint()是不会执行的**
上异步的runJob回调用下面这个方法，里面设置了JobWaiter，用来等待job执行完毕。
```scala
def runJob{
...
// job提交后会返回一个jobwaiter对象
val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
 waiter.completionFuture.value.get match {
​      case scala.util.Success(_) =>
​		...
}
```
### 2.1.4. **将Job放入队列**
给JOB分配一个ID，并将其放入队列，返回一个阻塞器，等待当前job执行完毕。将结果数据传送给handler function
```scala
def submitJob{
// 生成JOB的ID
val jobId = nextJobId.getAndIncrement()
// 生成阻塞器
val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
// post方法的实现：eventQueue.put(event),实际上是将此job提交到了一个LinkedBlockingDeque
eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      SerializationUtils.clone(properties)))
}
waiter
```
## 2.2. **job监听**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsA9AD.tmp.jpg) 
### 2.2.1. **监听触发**
在提交job时，我们将job放到了一个LinkedBlockingDeque队列，然后由EventLoop
负责接收处理请求，触发job的提交，产生一个finalStage.
EventLoop是在jobScheduler中启动的时候在JobGenerator中启动的
当从队列中拉去job时，开创建ResultStage:
```scala
class EventLoop
override def run(): Unit = {
​      try {
​        while (!stopped.get) {
​		 // 拉去job
​          val event = eventQueue.take()
​          try {
​			// 触发创建stage
​            onReceive(event)
​	...
}
def doOnReceive{
​	case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
​      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)
}
```
### 2.2.2. **初始化job和stage**
创建job：根据JobId，finalStage,excutor地址,job状态监听的JobListener,task的属性properties等生成job,并把job放入Map中记录。
```scala
// class DAGScheduler
private[scheduler] def handleJobSubmitted() {
// 以不同形式的hashMap存放job
 jobIdToStageIds = new HashMap[Int, HashSet[Int]]
stageIdToStage = new HashMap[Int, Stage]  
jobIdToActiveJob = new HashMap[Int, ActiveJob] 
// 初始化finalStage
var finalStage: ResultStage = 	createResultStage(finalRDD, func, partitions, jobId, callSite)
   // 初始化job
​    val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
​    clearCacheLocs()
​    jobIdToActiveJob(jobId) = job
​    activeJobs += job
​    finalStage.setActiveJob(job)
​    val stageIds = jobIdToStageIds(jobId).toArray
​    val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
​    **listenerBus.post(**
​      **SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))**
   // 提交finalStage，计算时会判断其之前是否存在shuffleStage，如果存在会优先计算shuffleStage，最后再计算finalStage
​    **submitStage(finalStage)**
  }
```
### 2.2.3. **提交stage**
**参见:** [MapOutputTrackerMaster](#_MapOutputTrackerMaster)
stage的状态分为三类：计算失败，计算完成和未计算完成，迭代的去计算完成父stage后，就可以到下一步，将stage转换到具体的task进行执行。
```scala
class DAGScheduler
private[scheduler] def handleJobSubmitted() {
 var finalStage: ResultStage = 	createResultStage(finalRDD, func, partitions, jobId, callSite)
​    ...
​    submitStage(finalStage)
  }
// 迭代的去判断父stage是否全部计算完成
private def submitStage(stage: Stage) {
if(jobId.isDefined){
val missing = getMissingParentStages(stage).sortBy(_.id)
if (missing.isEmpty) {
   // 父stage已经计算完成，可以开始当前计算
​	submitMissingTasks(stage, jobId.get)
​    } else {
​	//  父stage的map操作未完成，继续进行迭代
​          for (parent <- missing) {
​            submitStage(parent)
​          }
​          waitingStages += stage
​     }
}
}
// 获取未计算完成的stage
private def getMissingParentStages(stage: Stage): List[Stage] = {
...
for (dep <- rdd.dependencies) {
   dep match {
​      case shufDep: ShuffleDependency=>
// 判断当前stage是否计算完成
​	 val mapStage = getOrCreateShuffleMapStage(shufDep, stage.firstJobId)
​         if (!mapStage.isAvailable) {
​                  missing += mapStage
​          }           
​       case narrowDep: NarrowDependency[_] =>
​             waitingForVisit.push(narrowDep.rdd)
​      }
...
}
```
## 2.3. **stage转task**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsA9BE.tmp.jpg) 
首先利于上面说到的Stage知识获取所需要进行计算的task的分片;因为该Stage有些分片可能已经计算完成了;然后将Task运行依赖的RDD,Func,shuffleDep 进行序列化,通过broadcast发布出去; 然后创建Task对象,提交给taskScheduler调度器进行运行

### 2.3.1. **过滤需要执行的分片**
**参见:** [获取task分片](#___task__)
对Stage进行遍历所有需要运行的Task分片;
原因：存在部分task失败之类的情况,或者task运行结果所在的BlockManager被删除了,就需要针对特定分片进行重新计算;即所谓的恢复和重算机制;
```scala
class DAGScheduler{
def submitMissingTasks(stage, jobId){
​	val partitionsToCompute: Seq[Int] = stage.findMissingPartitions()
​	val properties = jobIdToActiveJob(jobId).properties
​	runningStages += stage
}
```
### 2.3.2. **序列化和广播**
对Stage的运行依赖进行序列化并broadcast给excutors(如果不序列化在数据传输过程中可能出错)
对ShuffleStage和FinalStage所序列化的内容有所不同：**对于ShuffleStage序列化的是RDD和shuffleDep;而对FinalStage序列化的是RDD和Func**
对于FinalStage我们知道,每个Task运行过程中,需要知道RDD和运行的函数,比如我们这里讨论的Count实现的Func;而对于ShuffleStage,ShuffleDependency记录了父RDD，排序方式，聚合器等，reduce端需要获取这些参数进行初始化和计算。
```scala
class DAGScheduler{
def submitMissingTasks(stage, jobId){
​	...
​      // consistent view of both variables.
RDDCheckpointData.synchronized {
​        taskBinaryBytes = stage match {
​          case stage: ShuffleMapStage =>
​            JavaUtils.bufferToArray(
​              closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
​          case stage: ResultStage =>           JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
​        }
​        partitions = stage.rdd.partitions
​      }
​      taskBinary = sc.broadcast(taskBinaryBytes)
```
### 2.3.3. **构造task对象**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsA9BF.tmp.jpg) 
针对每个需要计算的分片构造一个Task对象，
对于ResultTask就是在分片上调用我们的Func,而ShuffleMapTask按照ShuffleDep进行 MapOut

```scala
class DAGScheduler{
def submitMissingTasks(stage, jobId){
​	...
 val tasks: Seq[Task[_]] = try {
​      val serializedTaskMetrics = closureSerializer.serialize(stage.latestInfo.taskMetrics).array()
​      stage match {
​		// 一个stage会产生多个task任务
​        case stage: ShuffleMapStage =>
​          partitionsToCompute.map { id =>
​            new ShuffleMapTask()
​          }
​        case stage: ResultStage =>
​          partitionsToCompute.map { id =>
​            new ResultTask()
​          }
​      }
```
#### 2.3.3.1. **ShuffleMapTask**
#### 2.3.3.2. **ResultTask**
### 2.3.4. **taskScheduler调度task**
调用taskScheduler将task提交给Spark进行调度
```scala
class DAGScheduler{
def submitMissingTasks(stage, jobId){
	...
if (tasks.size > 0) {
   // 将taskSet发送给 taskScheduler
	taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptId, jobId, properties))
      stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
    } else {
     markStageAsFinished(stage, None)
    }
```
## 2.4. **获取运行结果**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsA9C0.tmp.jpg) 
DAGScheduler接收到DAGSchedulerEvent后判断其类型是TaskCompletion，不同的stage的实现方式不一样，shuffle的实现更复杂一点

```scala
 private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
  case completion: CompletionEvent =>
dagScheduler.handleTaskCompletion(completion)
}
private[scheduler] def handleTaskCompletion(event: CompletionEvent) {
event.reason match {
  case Success =>
​      task match {
​      case rt: ResultTask[_, _] =>
​			// 调用jobWaiter的taskSucced通知结果
​			job.listener.taskSucceeded
​	 case smt: ShuffleMapTask =>
​			// 调用outputTracker
​		mapOutputTracker.registerMapOutput
}
```
### 2.4.1. **ResultStage**
当计算完毕后，JobWaiter同步调用resultHandler处理task返回的结果。
```scala
private[scheduler] def handleTaskCompletion(event: CompletionEvent) {
event.reason match {
  case Success =>
      task match {
      case rt: ResultTask[_, _] =>
			// 调用jobWaiter的taskSucced通知结果
			job.listener.taskSucceeded(
			rt.outputId, event.result)
	 case smt: ShuffleMapTask =>
}
// jobWaiter是JobListner的子类
class JobWaiter extends JobListener{
  override def taskSucceeded(index: Int, result: Any): Unit = {
    synchronized {
      resultHandler(index, result.asInstanceOf[T])
    }
    if (finishedTasks.incrementAndGet() == totalTasks) {
      jobPromise.success(())
    }
  }
}
```
### 2.4.2. **ShuffleMapStage**
**参见:** [MapStatus的注册和获取](#_MapStatus______)
将运行结果(mapStatus)传送给outputTrancker
```scala
private[scheduler] def handleTaskCompletion(event: CompletionEvent) {
event.reason match {
  case Success =>
​      task match {
​      case rt: ResultTask[_, _] =>
​	 case smt: ShuffleMapTask =>
​			// 
​		mapOutputTracker.registerMapOutput(
​                shuffleStage.shuffleDep.shuffleId, 				smt.partitionId, status)
}
```
## 2.5. **doCheckPoint**
job执行完毕后执行
# 3. **stage**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsA9C1.tmp.jpg) 
1. 一组含有相同计算函数的任务集合，这些任务组合成了一个完整的job
2. stage分为两种：FinalStage和shuffleStage
3. stage中包含了jobId,对于FIFO规则，jobId越小的优先级越高
4. 为了保证容错性，一个stage可以被重复执行，所以在web UI上有可能看见多个stage的信息，取最新更新时间的即可
5. 组成：
```scala
private[scheduler] abstract class Stage(
    val id: Int, // stageId
    val rdd: RDD[_],// RDD that this stage runs on
    val numTasks: Int,// task数量
    val parents: List[Stage],// 父stage
    val firstJobId: Int,//当前stage上JobId
    val callSite: CallSite// 生成RDD存放位置
)  extends Logging {
```
## 3.1. **ShuffleMapStage** 
```scala
class ShuffleMapStage(
    val shuffleDep: ShuffleDependency[_, _, _],
    mapOutputTrackerMaster: MapOutputTrackerMaster)
  extends Stage(id, rdd, numTasks, parents, firstJobId, callSite){
// 判断当前stage是否可用
  def isAvailable: Boolean = numAvailableOutputs == numPartitions
}
```
### 3.1.1. **ShuffleMapTask**
每个运行在Executor上的Task, 通过SparkEnv获取shuffleManager对象, 然后调用getWriter来当前MapID=partitionId的一组Writer. 然后将rdd的迭代器传递给writer.write函数, 由每个Writer的实现去实现具体的write操作;
```scala
class ShuffleMapTask extends Task(
def runTask(context: TaskContext): MapStatus = {
​	// 反序列化接收到的数据
​    val (rdd, dep) = closureSerializer.deserialize(
​      ByteBuffer.wrap(taskBinary.value))
var writer: ShuffleWriter[Any, Any] = null
val manager = SparkEnv.get.shuffleManager
// 调用ShuffleManager的getWriter方法获取一组writer
 writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
// 遍历RDD进行write
 writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
writer.stop(success = true).get
}
}
```
上面代码中，在调用rdd的iterator()方法时，会根据RDD实现类的compute方法指定的处理逻辑对数据进行处理，当然，如果该Partition对应的数据已经处理过并存储在MemoryStore或DiskStore，直接通过BlockManager获取到对应的Block数据，而无需每次需要时重新计算。然后，write()方法会将已经处理过的Partition数据输出到磁盘文件。
在Spark Shuffle过程中，每个ShuffleMapTask会通过配置的ShuffleManager实现类对应的ShuffleManager对象（实际上是在SparkEnv中创建），根据已经注册的ShuffleHandle，获取到对应的ShuffleWriter对象，然后通过ShuffleWriter对象将Partition数据写入内存或文件。
### 3.1.2. **获取task分片**
**参见:** [过滤需要执行的分片](#__________)
返回需要计算的partition信息
```scala
class ShuffleMapStage{
def findMissingPartitions(): Seq[Int] = {
​    mapOutputTrackerMaster
​      .findMissingPartitions(shuffleDep.shuffleId)
​      .getOrElse(0 until numPartitions)
  }
}
```
## 3.2. **ResultStage** 
### 3.2.1. **ResultTask**
**参见:** [reduce端获取](#_reduce___)
ResultTask不需要进行写操作。直接将计算结果返回。
```scala
class ResultTask extends Task {
def runTask(context: TaskContext): U = {
	// 对RDD和函数进行反序列化
	val (rdd, func) = ser.deserialize(
      ByteBuffer.wrap(taskBinary.value)
	// 调用函数进行计算
	func(context, rdd.iterator(partition, context))
}
}
// RDD的iterator函数，
class RDD{
def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      getOrCompute(split, context)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
}
```
### 3.2.2. **获取task分片**
返回需要计算的partition信息,不需要经过tracker,在提交Job的时候会将其保存在ResultStage
```scala
class DAGScheduler{
def handleJobSubmitted(){
// 定义resultStage
finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
// 将job传递给resultStage
finalStage.setActiveJob(job)
}
}
class ResultStage{
// 过滤掉已经完成的
findMissingPartitions(): Seq[Int] = {
    val job = activeJob.get
    (0 until job.numPartitions).filter(id => !job.finished(id))
  }
}
```