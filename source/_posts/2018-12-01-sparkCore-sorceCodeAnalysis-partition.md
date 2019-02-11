---
title: sparkCore源码解析之partition
subtitle: 分区的源码解析
description: 水塘抽样
keywords: [spark,core,源码,partition]
author: liyz
date: 2019-01-14
tags: [spark,源码解析]
category: [spark]
---

 ![job](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/Partition.png)

# 1. **概念**

表示并行计算的一个计算单元

RDD 内部的数据集合在逻辑上和物理上被划分成多个小子集合，这样的每一个子集合我们将其称为分区，分区的个数会决定并行计算的粒度，而每一个分区数值的计算都是在一个单独的任务中进行，因此并行任务的个数，也是由 RDD（实际上是一个阶段的末 RDD，调度章节会介绍）分区的个数决定的
# 2. **获取分区** 
RDD的分区数量可通过rdd.getPartitions获取。
getPartitions方法是在RDD类中定义的，由不同的子类进行具体的实现
## 2.1. **接口**

**获取分区的定义**

在RDD类中定义了getPartition方法，返回一个Partition列表，Partition对象只含有一个编码index字段，不同RDD的partition会继承Partition类，例如JdbcPartition、KafkaRDDPartition，HadoopPartition等。

```scala
class RDD{
	// 获取分区定义
	def getPartitions:Array[Partition]
}
 
// Partition类定义
trait Partition {
	def index:Int
}
```
## 2.2. **实现**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsC537.tmp.jpg) 

transformation类型的RDD分区数量和父RDD的保持一致，Action类型的RDD分区数量，不同的数据源默认值不一样，获取的方式也不同
### 2.2.1. **KafkaRDD**

kafkaRDD的partition数量等于compute方法中生成OffsetRange的数量。

```scala
// DirectKafkaInputDStream类在接收到消息后通过compute方法计算得到OffsetRange
class OffsetRange( 
	val topic:String, // Kafka topic name
	val partition:Int, // Kafka partition id
	val fromOffset:Long,
	val untilOffset:Long
){...}
 
class KafkaRDD(
		val offsetRages:Array[OffsetRange]
	) extends RDD{
	// 遍历OffsetRange数组，得到数组下标和偏移量等信息生成KafkaRDDPartition
	offsetRanges.zipWithIndex.map{
		case(o,i)=>
			new KafkaRDDPartition(
				i,o.topic,o.partition,
				o.fromOffset,o.untilOffset
			)
	}.toArray
}
```
### 2.2.2. **HadoopRDD**

HadoopRDD的分区是基于hadoop的splits方法进行的。每个partition的大小默认等于hdfs的block的大小

例如：一个txt文件12800M,则
`val rdd1=sc.textFile("/data.txt");`
rdd1默认会有12800/128=10个分区。


```scala
class HadoopRDD(){
	
	// 生成一个RDD的唯一ID
	val id=Int=sc.newRddId()
 
	def getPartitions:Array[Partition]={
		// 调用hadoop的splits方法进行切割
		val inputSplits=inputFormat.getSplits(
			jobConf,minPartitions
		)
		// 组成spark的partition
		val array=new Array[Partition](inputSplits.size)
		for(i <- 0 until inputSplits.size){
			array(i)=new HadoopPartition(id,i,
				inputSplits(i))
		}
	}
}
```


hadoop的FileInputFormat类：
texfile的分区大小时指定的分区数和block树中取较大值，所以当指定numPartitions，小于block数时无效，大于则生效
### 2.2.3. **JdbcRDD**

JDBC的partition划分是指定开始行和结束行，然后将查询到的结果分为3个（默认值）partition。

```scala
class JdbcRDD(numPartitions:Int){
	def getPartitions:Array[Partition]={
		(0 until numPartitions).map{
			new JdbcPartition(i,start,end)
		}.toArray
	}
}
```

### 2.2.4. **MapPartitionsRDD**

转换类的RDD分区数量是由其父类的分区数决定的

```scala
// 获取父RDD列表的第一个RDD
class RDD{
	def firstParent:RDD={
		dependencies.head.rdd.asInstanceOf[RDD]
	}
}
class MapPartitionsRDD(){
	// 获取父RDD的partitions数量
	def getPartitions: Array[Partition] = 			firstParent[T].partitions
}
```
# 3. **分区数量**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsC557.tmp.jpg) 

分区数量的原则：尽可能的选择大的分区值
## 3.1. **RDD初始化相关**

|Spark API|partition数量|
|---|---|
|sc.parallelize(…)	|	sc.defaultParallelism|
|sc.textFile(…)		|	max(传参, block数)|
|sc.newAPIHadoopRDD(…)|	max(传参, block数)|
|new JdbcRDD(…)	|传参|

## 3.2. **通用transformation**

- filter(),map(),flatMap(),distinct()：和父RDD相同
- union： 两个RDD的和rdd.union(otherRDD)：rdd.partitions.size + otherRDD. partitions.size
- intersection：取较大的rdd.intersection(otherRDD)：max(rdd.partitions.size, otherRDD. partitions.size)
- rdd.subtract(otherRDD)	：rdd.partitions.size
- cartesian：两个RDD数量的乘积rdd.cartesian(otherRDD)：
  rdd.partitions.size * otherRDD. partitions.size


## 3.3. **Key-based Transformations**

reduceByKey(),foldByKey(),combineByKey(), groupByKey(),sortByKey(),mapValues(),flatMapValues()	和父RDD相同

cogroup(), join(), ,leftOuterJoin(), rightOuterJoin():
所有父RDD按照其partition数降序排列，从partition数最大的RDD开始查找是否存在partitioner，存在则partition数由此partitioner确定，否则，所有RDD不存在partitioner，由spark.default.parallelism确定，若还没设置，最后partition数为所有RDD中partition数的最大值

# 4. **分区器** 
**注意：只有Key-Value类型的RDD才有分区的，非Key-Value类型的RDD分区的值是None的**

```scala
abstract class Partitioner extends Serializable {
  def numPartitions: Int // 分区数量
  def getPartition(key: Any): Int // 分区编号
}
```

## 4.1. **作用**

partitioner分区器作用：

1. 决定Shuffle过程中Reducer个数（实际上是子RDD的分区个数）以及Map端一条数据记录应该分配给那几个Reducer
2. 决定RDD的分区数量，例如执行groupByKey(new HashPartitioner(2))所生成的ShuffledRDD中，分区数目等于2
3. 决定CoGroupedRDD与父RDD之间的依赖关系

## 4.2. **种类**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsC559.tmp.jpg) 

分区器的选择：

1. 如果RDD已经有了分区器，则在已有分区器里面挑选分区数量最多的一个分区器。
2. 如果RDD没有指定分区器，则默认使用HashPartitioner分区器。
3. 用户可以自己声明RangePartitioner分区器

```scala

object Partitioner{
	def defaultPartitioner(rdd):Partitioner={
		val hasPartitioner=
			rdds.filter(
				_.partitioner.exists(_numPartitions>0))
			}
		// 如果RDD已经有分区则选取其分区数最多的
		if(hasPartitioner.nonEmpty){
			hasPartitioner.maxBy(_.partitions.length).
				partitioner.get
		}else{
			if(rdd.context.conf.contains(
				"spark.default.parallelism"
			)){
				// 如果在conf中配置了分区数则用之
				new HashPartitioner(
					rdd.context.defaultParallelism
				)
			}else{
				// 如果没有配置parallelism则和父RDD中最大的保持一致
				new HashPartitioner(rdds.map(
					_.partitions.length
				).max)
			}
		}
}

```

### 4.2.1. **HashPartitioner**

HashPartitioner分区的原理很简单，对于给定的key，计算其hashCode，并除于分区的个数取余，**如果余数小于0，则用余数+分区的个数，最后返回的值就是这个key所属的分区ID**

```scala
class HashPartitioner(partitions:Int) {
	def getPartition(key:Any):Int=key match{
		case null=> 0
		case _=> nonNegativeMod(key.hashCode, numPartitions)
	}
  def nonNegativeMod(x: Int, mod: Int): Int = {
    val rawMod = x % mod
    rawMod + (if (rawMod < 0) mod else 0)
  }	
	// 判断两个RDD分区方式是否一样
	def equals(other:Any):Boolean= other match{
		case h:HashPartitioner => 
			h.numPartitions==numPartitions
		case  _ => false
	}
}
```
### 4.2.2. **RangePartitioner**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsC55A.tmp.jpg) 

 

HashPartitioner分区可能导致每个分区中数据量的不均匀。而RangePartitioner分区则尽量保证每个分区中数据量的均匀，将一定范围内的数映射到某一个分区内。分区与分区之间数据是有序的，但分区内的元素是不能保证顺序的。

  RangePartitioner分区执行原理：

1. 计算总体的数据抽样大小sampleSize，计算规则是：至少每个分区抽取20个数据或者最多1M的数据量。
2. 根据sampleSize和分区数量计算每个分区的数据抽样样本数量最大值sampleSizePrePartition
3. 根据以上两个值进行水塘抽样，返回RDD的总数据量，分区ID和每个分区的采样数据。
4. 计算出数据量较大的分区通过RDD.sample进行重新抽样。
5. 通过抽样数组 candidates: ArrayBuffer[(K, wiegth)]计算出分区边界的数组BoundsArray
6. 在取数据时，如果分区数小于128则直接获取，如果大于128则通过二分法，获取当前Key属于那个区间，返回对应的BoundsArray下标即为partitionsID

一句话概括：**就是遍历每个paritiion，对里面的数据进行抽样，把抽样的数据进行排序，并按照对应的权重确定边界**
#### 4.2.2.1. **获取区间数组**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsC56B.tmp.jpg) 
##### 4.2.2.1.1. **给定样本总数**

给定总的数据抽样大小，最多1M的数据量(10^6)，最少20倍的RDD分区数量，也就是每个RDD分区至少抽取20条数据
​    
```scala
class RangePartitioner(partitions,rdd) {
// 1. 计算样本大小
 val sampleSize =math.min(20.0 * partitions, 1e6)
}
```
##### 4.2.2.1.2. **计算样本最大值**

RDD各分区中的数据量可能会出现倾斜的情况，乘于3的目的就是保证数据量小的分区能够采样到足够的数据，而对于数据量大的分区会进行第二次采样

```scala
class RangePartitioner(partitions,rdd) {
​	
// 1. 计算样本大小
 val sampleSize = 
​			math.min(20.0 * partitions, 1e6)
 
// 2. 计算样本最大值
val sampleSizePerPartition = 
​	math.ceil(
​		3.0 * sampleSize / rdd.partitions.length
​	).toInt
}
```
##### 4.2.2.1.3. **水塘抽样**

根据以上两个值进行水塘抽样，返回RDD的总数据量，分区ID和每个分区的采样数据。其中总数据量通过遍历RDD所有partition的key累加得到的，不是通过rdd.count计算得到的

```scala
class RangePartitioner(partitions,rdd) {
	
// 1. 计算样本大小
 val sampleSize =math.min(20.0 * partitions, 1e6)
 
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(
		3.0 * sampleSize / rdd.partitions.length
	).toInt
}
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = RangePartitioner.sketch(
		rdd.map(_._1), sampleSizePerPartition)
 
```


##### 4.2.2.1.4. **是否需要二次采样**

如果有较大RDD存在，则按照平均值去采样的话数据量太少，容易造成数据倾斜，所以需要进行二次采样

判断是否需要重新采样方法：
样本数量占比乘以当前RDD的总行数大于预设的每个RDD最大抽取数量，说明这个RDD的数据量比较大，需要采样更多的数据：eg: 0.2*100=20<60;0.2*20000=2000>60

```scala
class RangePartitioner(partitions,rdd) {
	
// 1. 计算样本大小
 val sampleSize =math.min(20.0 * partitions, 1e6)
 
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(
		3.0 * sampleSize / rdd.partitions.length
	).toInt
}
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = RangePartitioner.sketch(
		rdd.map(_._1), sampleSizePerPartition)
 
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
 if (fraction * n > sampleSizePerPartition) {
	// 记录需要重新采样的RDD的ID
	imbalancedPartitions += idx 
}
```
##### 4.2.2.1.5. **计算样本权重**

计算每个采样数据的权重占比，根据采样数据的ID和权重生成出RDD分区边界数组

权重计算方法：总数据量/当前RDD的采样数据量

```scala
class RangePartitioner(partitions,rdd) {
​	
// 1. 计算样本大小
 val sampleSize = 
​			math.min(20.0 * partitions, 1e6)
 
// 2. 计算样本最大值
val sampleSizePerPartition = 
​	math.ceil(
​		3.0 * sampleSize / rdd.partitions.length
​	).toInt
}
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = 		RangePartitioner.sketch(
​		rdd.map(_._1), sampleSizePerPartition)
 
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
 
//  5. 保存样本数据的集合buffer:包含数据和权重
val candidates = ArrayBuffer.empty[(K, Float)]
 
 if (fraction * n > sampleSizePerPartition) {
​	// 记录需要重新采样的RDD的ID
​	imbalancedPartitions += idx 
}else{
// 5. 计算样本权重
​	val weight = (
​	  // 采样数据的占比
​		n.toDouble / sample.length).toFloat 
​            for (key <- sample) {
​			// 记录采样数据key和权重
​              candidates += ((key, weight))
​            }
​	}
}
```
##### 4.2.2.1.6. **二次抽样**

 对于数据分布不均衡的RDD分区，重新进行二次抽样。
二次抽样采用的是RDD的采样方法：RDD.sample

```scala
class RangePartitioner(partitions,rdd) {
​	
// 1. 计算样本大小
 val sampleSize = 
​			math.min(20.0 * partitions, 1e6)
 
// 2. 计算样本最大值
val sampleSizePerPartition = 
​	math.ceil(
​		3.0 * sampleSize / rdd.partitions.length
​	).toInt
}
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = 		RangePartitioner.sketch(
​		rdd.map(_._1), sampleSizePerPartition)
 
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
 
//  5. 保存样本数据的集合buffer:包含数据和权重
val candidates = ArrayBuffer.empty[(K, Float)]
 
 if (fraction * n > sampleSizePerPartition) {
​	// 记录需要重新采样的RDD的ID
​	imbalancedPartitions += idx 
}else{
// 5. 计算样本权重
​	val weight = (
​	  // 采样数据的占比
​		n.toDouble / sample.length).toFloat 
​            for (key <- sample) {
​			// 记录采样数据key和权重
​              candidates += ((key, weight))
​            }
​	}
// 6. 对于数据分布不均衡的RDD分区，重新数据抽样
if (imbalancedPartitions.nonEmpty) {
​	// 利用rdd的sample抽样函数API进行数据抽样
​          val reSampled = imbalanced.sample(
​				withReplacement = 
​					false, fraction, seed).collect()
}
 
}
```
##### 4.2.2.1.7. **生成边界数组**

将最终的抽样数据计算出分区边界数组返回，边界数组里面存放的是RDD里面数据的key值，
比如最终返回的数组是：array[0,10,20,30..]
其中0,10,20,30是采样数据中的key值，对于每一条数据都会判断其在此数组的那个区间中间，例如有一条数据key值是3则其在0到10之间，属于第一个分区，同理Key值为15的数据在第二个分区

```scala
class RangePartitioner(partitions,rdd) {
​	
// 1. 计算样本大小
 val sampleSize = 
​			math.min(20.0 * partitions, 1e6)
 
// 2. 计算样本最大值
val sampleSizePerPartition = 
​	math.ceil(
​		3.0 * sampleSize / rdd.partitions.length
​	).toInt
}
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = 		RangePartitioner.sketch(
​		rdd.map(_._1), sampleSizePerPartition)
 
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
 
//  5. 保存样本数据的集合buffer:包含数据和权重
val candidates = ArrayBuffer.empty[(K, Float)]
 
 if (fraction * n > sampleSizePerPartition) {
​	// 记录需要重新采样的RDD的ID
​	imbalancedPartitions += idx 
}else{
// 5. 计算样本权重
​	val weight = (
​	  // 采样数据的占比
​		n.toDouble / sample.length).toFloat 
​            for (key <- sample) {
​			// 记录采样数据key和权重
​              candidates += ((key, weight))
​            }
​	}
// 6. 对于数据分布不均衡的RDD分区，重新数据抽样
if (imbalancedPartitions.nonEmpty) {
​	// 利用rdd的sample抽样函数API进行数据抽样
​          val reSampled = imbalanced.sample(
​				withReplacement = 
​					false, fraction, seed).collect()
}
// 7. 生成边界数组
RangePartitioner.determineBounds(
​	candidates, partitions)
}
```
#### 4.2.2.2. **水塘抽样算法**

水塘抽样概念：
它是一系列的随机算法，其目的在于从包含n个项目的集合S中选取k个样本，使得每条数据抽中的概率是k/n。其中n为一很大或未知的数量，尤其适用于不能把所有n个项目都存放到主内存的情况

我们可以：定义取出的行号为choice，第一次直接以第一行作为取出行 choice ，而后第二次以二分之一概率决定是否用第二行替换 choice ，第三次以三分之一的概率决定是否以第三行替换 choice ……，以此类推。由上面的分析我们可以得出结论，在取第n个数据的时候，我们生成一个0到1的随机数p，如果p小于1/n，保留第n个数。大于1/n，继续保留前面的数。直到数据流结束，返回此数，算法结束。

详见：
<https://www.iteblog.com/archives/1525.html>
<https://my.oschina.net/freelili/blog/2987667>

 实现：
1. 获取到需要抽样RDD分区的样本大小k和分区的所有KEY数组input
2. 初始化抽样结果集reservoir为分区前K个KEY值
3. 如果分区的总数小于预计样本大小k,则将当前分区的所有数据作为样本数据，否则到第四步
4. 遍历分区里所有Key组成的数组input
5. 生成随机需要替换input数组的下标，如果下标小于K则替换
6. 返回抽取的key值数组和当前分区的总数据量： (reservoir, l)

难点：
```scala
 // 计算出需要替换的数组下标
// 选取第n个数的概率是：n/l; 如果随机替换数组值的概率是p=rand.nextDouble，
// 则如果p<k/l;则替换池中任意一个数，即： p*l < k 则进行替换，用p*l作为随机替换的下标
 val replacementIndex = (rand.nextDouble() * l).toLong
if (replacementIndex < k) {
// 替换reservoir[随机抽取的下标]的值为input[l]的值item
          reservoir(replacementIndex.toInt) = item
 }
```
#### 4.2.2.3. **定位分区ID**
![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/sparkcore/wpsC56C.tmp.jpg) 

如果分区边界数组的大小小于或等于128的时候直接变量数组，否则采用二分查找法确定key属于某个分区。


##### 4.2.2.3.1. **数组直接获取**

遍历数组，判断当前key值是否属于当前区间

```scala
// 根据RDD的key值返回对应的分区id。从0开始
def getPartition(key: Any): Int = {
​    // 强制转换key类型为RDD中原本的数据类型
​    val k = key.asInstanceOf[K]
​    var partition = 0
​    if (rangeBounds.length <= 128) {
​      // 如果分区数据小于等于128个，那么直接本地循环寻找当前k所属的分区下标
​      // ordering.gt(x,y):如果x>y,则返回true
​      while (partition < rangeBounds.length && ordering.gt(k, rangeBounds(partition))) {
​        partition += 1
​      }
 
```
##### 4.2.2.3.2. **二分法查找**

对于分区数大于128的情况，采样二分法查找
 ```scala
 // 根据RDD的key值返回对应的分区id。从0开始
  def getPartition(key: Any): Int = {
 // 如果分区数量大于128个，那么使用二分查找方法寻找对应k所属的下标;
​      // 但是如果k在rangeBounds中没有出现，实质上返回的是一个负数(范围)或者是一个超过rangeBounds大小的数(最后一个分区，比所有数据都大)
​      // Determine which binary search method to use only once.
​      partition = binarySearch(rangeBounds, k)
​      // binarySearch either returns the match location or -[insertion point]-1
​      if (partition < 0) {
​        partition = -partition-1
​      }
​      if (partition > rangeBounds.length) {
​        partition = rangeBounds.length
​      }
 ```
# 5. **自定义分区器**

自定义：
1. 继承Partitioner方法，
2. 重写getPartition、numPartitions、equals等方法。
```scala
public class MyPartioner extends Partitioner { 
    @Override 
    public int numPartitions() { 
        return 1000; 
    } 

    @Override 
    public int getPartition(Object key) { 
        String k = (String) key; 
        int code = k.hashCode() % 1000; 
        System.out.println(k+":"+code); 
        return  code < 0?code+1000:code; 
    } 

    @Override 
    public boolean equals(Object obj) { 
        if(obj instanceof MyPartioner){ 
            if(this.numPartitions()==((MyPartioner) obj).numPartitions()){ 
                return true; 
            } 
            return false; 
        } 
        return super.equals(obj); 
    } 
} 
```

调用：`pairRdd.groupbykey(new MyPartitioner())`



参考链接：<https://ihainan.gitbooks.io/spark-source-code/content/section1/rddPartitions.html>