---
title: sparkCore源码解析之block
subtitle: RDD组成之block源码分析
keywords: [spark,core,源码,block]
author: liyz
date: 2019-01-14
tags: [spark,源码解析]
category: [spark]
---
![block](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/Block.png)
#  1. **标记**
Apache Spark中，对Block的查询、存储管理，是通过唯一的Block ID来进行区分的。

同一个Spark Application，以及多个运行的Application之间，对应的Block都具有唯一的ID
## 1.1. **种类**

需要在worker和driver间共享数据时，就需要对这个数据进行唯一的标识，常用的需要传输的block信息有以下几类
RDDBlockId、ShuffleBlockId、ShuffleDataBlockId、ShuffleIndexBlockId、BroadcastBlockId、TaskResultBlockId、TempLocalBlockId、TempShuffleBlockId

## 1.2. **生成规则**


```scala
RDDBlockId : "rdd_" + rddId + "_" + splitIndex
ShuffleBlockId : "shuffle_" + shuffleId + "_" + mapId + "_" + reduceId
ShuffleDataBlockId:"shuffle_" + shuffleId + "_" + mapId + "_" + reduceId + ".data"
ShuffleIndexBlockId:"shuffle_" + shuffleId + "_" + mapId + "_" + reduceId + ".index"
TaskResultBlockId:"taskresult_" + taskId
StreamBlockId:"input-" + streamId + "-" + uniqueId
...
```
# 2. **存储**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wps6E0A.tmp.jpg) 

DiskStore是通过DiskBlockManager进行管理存储到磁盘上的Block数据文件的，在同一个节点上的多个Executor共享相同的磁盘文件路径，相同的Block数据文件也就会被同一个节点上的多个Executor所共享。而对应MemoryStore，因为每个Executor对应独立的JVM实例，从而具有独立的Storage/Execution内存管理，所以使用MemoryStore不能共享同一个Block数据，但是同一个节点上的多个Executor之间的MemoryStore之间拷贝数据，比跨网络传输要高效的多
## 2.1. **MemoryStore**

数据在内存中存储的形式
1. 以序列化格式
2. 以反序列化的形式
	​	2.1 Block数据记录能够完全放到内存中
	​	2.2 Block数据记录只能部分放到内存中：申请Unroll内存（预占内存）
3. 以序列化二进制格式保存Block数据

```scala
MEMORY_ONLY
MEMORY_ONLY_2
MEMORY_ONLY_SER
MEMORY_ONLY_SER_2
MEMORY_AND_DISK
MEMORY_AND_DISK_2
MEMORY_AND_DISK_SER
MEMORY_AND_DISK_SER_2
OFF_HEAP
```
## 2.2. **DiskStore**
数据罗盘的几种形式：

1. 通过文件流写Block数据
2. 将二进制Block数据写入文件
3. 
```scala
DISK_ONLY
DISK_ONLY_2
MEMORY_AND_DISK
MEMORY_AND_DISK_2
MEMORY_AND_DISK_SER
MEMORY_AND_DISK_SER_2
OFF_HEAP
```
### 2.2.1. **DiskBlockManager**

DiskStore即基于文件来存储Block. 基于Disk来存储,首先必须要解决一个问题就是磁盘文件的管理:磁盘目录结构的组成,目录的清理等,在Spark对磁盘文件的管理是通过 DiskBlockManager来进行管理的

DiskBlockManager管理了每个Block数据存储位置的信息，包括从Block ID到磁盘上文件的映射关系。DiskBlockManager主要有如下几个功能：

1. 负责创建一个本地节点上的指定磁盘目录，用来存储Block数据到指定文件中
2. 如果Block数据想要落盘，需要通过调用getFile方法来分配一个唯一的文件路径
3. 如果想要查询一个Block是否在磁盘上，通过调用containsBlock方法来查询
4. 查询当前节点上管理的全部Block文件
通过调用createTempLocalBlock方法，生成一个唯一Block ID，并创建一个唯一的临时文件，用来存储中间结果数据
5. 通过调用createTempShuffleBlock方法，生成一个唯一Block ID，并创建一个唯一的临时文件，用来存储Shuffle过程的中间结果数据
## 2.3. **offHeap**


堆外存储不支持序列化和副本


Spark中实现的OffHeap是基于Tachyon:分布式内存文件系统来实现的
# 3. **内存管理模型**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wps6E0C.tmp.jpg) 

在Spark Application提交以后，最终会在Worker上启动独立的Executor JVM，Task就运行在Executor里面。在一个Executor JVM内部，内存管理模型就是管理excutor运行所需要的内存

<http://shiyanjun.cn/archives/1585.html>

## 3.1. **StaticMemoryManager**

1.5之前版本使用
缺点：

1. 没有一个合理的默认值能够适应不同计算场景下的Workload
2. 内存调优困难，需要对Spark内部原理非常熟悉才能做好
3. 对不需要Cache的Application的计算场景，只能使用很少一部分内存
## 3.2. **UnifiedMemoryManager**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wps6E0D.tmp.jpg) 

统一内存分配管理模型：
1. 可以动态的分配excution和storage的内存大小
2. 不仅可以分配堆内内存，也可以分配堆外内存
3. 堆外内存和分配比例都可以通过参数配置
4. 内存的分配和回收是通过MemoryPool控制

```scala
abstract class MemoryManager(
​    conf: SparkConf,
​    numCores: Int,
​    onHeapStorageMemory: Long,
​    onHeapExecutionMemory: Long){
​	
 // storage堆内内存
  @GuardedBy("this")
  protected val onHeapStorageMemoryPool = 		new StorageMemoryPool(this, 		MemoryMode.ON_HEAP)
 // storage堆外内存
  @GuardedBy("this")
  protected val offHeapStorageMemoryPool = 		new StorageMemoryPool(this, 		MemoryMode.OFF_HEAP)
// execution堆内内存
  @GuardedBy("this")
  protected val onHeapExecutionMemoryPool = 		new ExecutionMemoryPool(this, 		MemoryMode.ON_HEAP)
// excution堆外内存
  @GuardedBy("this")
  protected val offHeapExecutionMemoryPool = 		new ExecutionMemoryPool(this, 		MemoryMode.OFF_HEAP)
}
// 默认最大堆内存
val maxOffHeapMemory = conf.getSizeAsBytes("spark.memory.offHeap.size", 0)
 
// 默认storage和excution的内存大小各占50%
offHeapStorageMemory =
​    (maxOffHeapMemory * conf.getDouble("spark.memory.storageFraction", 0.5)).toLong
 
```
### 3.2.1. **内存划分**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wps6E1E.tmp.jpg) 

在统一内存管理模型中，storage和excution内存大小可以动态调整，在一定程度上减少了OOM发生概率


默认内存划分：

预留内存reservedMemory=300M
管理内存maxHeapMemory = (systemMemory - reservedMemory) * 0.6
storageMemory=excutionMemory=maxHeapMemory*0.5

非堆内存默认值0，可通过spark.memory.offHeap.size参数调整，其中storage和excution的内存占比也均为50%

#### 3.2.1.1. **Storage内存区**

Storage内存，用来缓存Task数据、在Spark集群中传输（Propagation）内部数据
#### 3.2.1.2. **Execution内存区**

Execution内存，用于满足Shuffle、Join、Sort、Aggregation计算过程中对内存的需求
#### 3.2.1.3. **预留内存**
#### 3.2.1.4. **非堆内存**
### 3.2.2. **内存调控**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wps6E2E.tmp.jpg) 
#### 3.2.2.1. **Storage内存**
##### 3.2.2.1.1. **申请**

1. 判断申请内存类型：堆内还是堆外
2. 如果申请内存大于剩余内存总量则申请失败
3. 如果申请内存大小在storage内存范围内则直接分配
4. 如果申请内存大于storage剩余内存则借用excution内存

```scala
 
// 为blockId申请numBytes字节大小的内存
override def acquireStorageMemory ()synchronized { 
  val (executionPool, storagePool, maxMemory) = memoryMode match { 
// 根据memoryMode值，返回对应的StorageMemoryPool与ExecutionMemoryPool
​    case MemoryMode.ON_HEAP =>
​    case MemoryMode.OFF_HEAP => 
  }
  if (numBytes > maxMemory) {
 // 申请的内存大于剩余内存总理则申请失败
​      s"memory limit ($maxMemory bytes)")
​    return false
  }
  if (numBytes > storagePool.memoryFree) { 
// 如果Storage内存块中没有足够可用内存给blockId使用，则计算当前Storage内存区缺少多少内存，然后从Execution内存区中借用
​    val memoryBorrowedFromExecution = Math.min(executionPool.memoryFree, numBytes)
// Execution内存区减掉借用内存量
executionPool.decrementPoolSize(memoryBorrowedFromExecution) 
// Storage内存区增加借用内存量    storagePool.incrementPoolSize(memoryBorrowedFromExecution) 
  }
// 如果Storage内存区可以为blockId分配内存，直接成功分配；否则，如果从Execution内存区中借用的内存能够满足blockId，则分配成功，不能满足则分配失败。
  storagePool.acquireMemory(blockId, numBytes) 
}
```
##### 3.2.2.1.2. **释放**

释放Storage内存比较简单，只需要更新Storage内存计量变量即可

```scala
def releaseMemory(size: Long): Unit = lock.synchronized {
  if (size > _memoryUsed) {
​    // 需要释放内存大于已使用内存，则直接清零
​    _memoryUsed = 0
  } else {
​	// 从已使用内存中减去释放内存大小
​    _memoryUsed -= size
  }
}
```
#### 3.2.2.2. **Excution内存** 
excution内存的获取和释放都是线程安全的，而且分配给每个task的内存大小是均等的，每当有task运行完毕后，都会触发内存的回收操作。
##### 3.2.2.2.1. **申请**

如果从storage申请内存大小比storage剩余内存大，则申请线程会阻塞，并对storage内存发起缩小操作。直到storage释放足够内存。

Execution内存区内存分配的基本原则：
如果有N个活跃（Active）的Task在运行，ExecutionMemoryPool需要保证每个Task在将中间结果数据Spill到磁盘之前，至少能够申请到当前Execution内存区对应的Pool中1/2N大小的内存量，至多是1/N大小的内存。

这里N是动态变化的，因为可能有新的Task被启动，也有可能Task运行完成释放资源，所以ExecutionMemoryPool会持续跟踪ExecutionMemoryPool内部Task集合memoryForTask的变化，并不断地重新计算分配给每个Task的这两个内存量的值：1/2N和1/N。
##### 3.2.2.2.2. **释放**

```scala
// 同步的释放内存
def releaseMemory(numBytes: Long, taskAttemptId: Long): Unit = lock.synchronized {
  val curMem = memoryForTask.getOrElse(taskAttemptId, 0L)
// 计算释放内存大小
 var memoryToFree = if (curMem < numBytes) { 
​	// 没有足够内存需要释放，则释放掉当前task所有使用内存
​    curMem
  } else {
​    numBytes
  }
  if (memoryForTask.contains(taskAttemptId)) { // Task执行完成，从内部维护的memoryForTask中移除
​    memoryForTask(taskAttemptId) -= memoryToFree
​    if (memoryForTask(taskAttemptId) <= 0) {
​      memoryForTask.remove(taskAttemptId)
​    }
  }
 // 通知调用acquireMemory()方法申请内存的Task内存已经释放
  lock.notifyAll()
}
```
# 4. **BlockManager**
BlockManagerMaster管理BlockManager.
BlockManager在每个Dirver和Executor上都有，用来管理Block数据，包括数据的获取和保存等

谈到Spark中的Block数据存储，我们很容易能够想到BlockManager，他负责管理在每个Dirver和Executor上的Block数据，可能是本地或者远程的。具体操作包括查询Block、将Block保存在指定的存储中，如内存、磁盘、堆外（Off-heap）。而BlockManager依赖的后端，对Block数据进行内存、磁盘存储访问，都是基于前面讲到的MemoryStore、DiskStore。
在Spark集群中，当提交一个Application执行时，该Application对应的Driver以及所有的Executor上，都存在一个BlockManager、BlockManagerMaster，而BlockManagerMaster是负责管理各个BlockManager之间通信，这个BlockManager管理集群


## 4.1. **读数据**

每个Executor上都有一个BlockManager实例，负责管理用户提交的该Application计算过程中产生的Block。

很有可能当前Executor上存储在RDD对应Partition的经过处理后得到的Block数据，也有可能当前Executor上没有，但是其他Executor上已经处理过并缓存了Block数据，所以对应着本地获取、远程获取两种可能
## 4.2. **BlockManager集群**

关于一个Application运行过程中Block的管理，主要是基于该Application所关联的一个Driver和多个Executor构建了一个Block管理集群：Driver上的(BlockManagerMaster, BlockManagerMasterEndpoint)是集群的Master角色，所有Executor上的(BlockManagerMaster, RpcEndpointRef)作为集群的Slave角色。当Executor上的Task运行时，会查询对应的RDD的某个Partition对应的Block数据是否处理过，这个过程中会触发多个BlockManager之间的通信交互
## 4.3. **状态管理**

BlockManager在进行put操作后，通过blockInfoManager来控制当前put等操作是否完成以及是否成功。

对于BlockManager中的存储的每个Block,不一定是对应的数据都PUT成功了,不一定可以立即提供对外的读取,因为PUT是一个过程,有成功还是有失败的状态. ,拿ShuffleBlock来说,在shuffleMapTask需要Put一个Block到BlockManager中,在Put完成之前,该Block将处于Pending状态,等待Put完成了不代表Block就可以被读取, 因为Block还可能Put"fail"了.

因此BlockManager通过BlockInfo来维护每个Block状态,在BlockManager的代码中就是通过一个TimeStampedHashMap来维护BlockID和BlockInfo之间的map.

private val blockInfo = new TimeStampedHashMap[BlockId, BlockInfo]
注： 2.2中此处是通过线程安全的hashMap和一个计数器实现的 
# 5. **读写控制**

BlockInfoManager通过同步机制防止多个task处理同一个block数据块

用户提交一个Spark Application程序，如果程序对应的DAG图相对复杂，其中很多Task计算的结果Block数据都有可能被重复使用，这种情况下如何去控制某个Executor上的Task线程去读写Block数据呢？其实，BlockInfoManager就是用来控制Block数据读写操作，并且跟踪Task读写了哪些Block数据的映射关系，这样如果两个Task都想去处理同一个RDD的同一个Partition数据，如果没有锁来控制，很可能两个Task都会计算并写同一个Block数据，从而造成混乱

```scala
class BlockInfoManager{
	val infos = 
		new mutable.HashMap[BlockId, BlockInfo]
	// 存放被锁定任务列表
	val writeLocksByTask =
    	new mutable.HashMap[
			TaskAttemptId, mutable.Set[BlockId]]
	 val readLocksByTask =
    	new mutable.HashMap[TaskAttemptId, 		ConcurrentHashMultiset[BlockId]]
 
	def lockForReading(）{
		infos.get(blockId) match {
			case Some(info) =>
				// 没有写任务
          		if (info.writerTask == 					BlockInfo.NO_WRITER) {
					// 读task数量加一
            		info.readerCount += 1
					//	放入读多锁定队列
 					readLocksByTask(
					currentTaskAttemptId).
					add(blockId)
	}
}
def lockForWriting(）{
		case Some(info) =>
          if (info.writerTask == 				BlockInfo.NO_WRITER && 				info.readerCount == 0) {
            info.writerTask = currentTaskAttemptId
 			writeLocksByTask.addBinding(
			currentTaskAttemptId, blockId)
}
 
```

