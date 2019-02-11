---
title: spark源码解析之RangePartitioner
subtitle: spark rdd 分区器
description: ,RangePartitioner详解
keywords: [水塘抽样,RangePartitioner,分区]
date: 2018-12-10
tags: [spark,源码解析]
category: [spark]
---



![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/wps37A2.tmp.jpg)

 

HashPartitioner分区可能导致每个分区中数据量的不均匀。而RangePartitioner分区则尽量保证每个分区中数据量的均匀，将一定范围内的数映射到某一个分区内。分区与分区之间数据是有序的，但分区内的元素是不能保证顺序的。

   RangePartitioner分区执行原理：

1. 计算总体的数据抽样大小sampleSize，计算规则是：至少每个分区抽取20个数据或者最多1M的数据量。

2. 根据sampleSize和分区数量计算每个分区的数据抽样样本数量最大值sampleSizePrePartition

3. 根据以上两个值进行水塘抽样，返回RDD的总数据量，分区ID和每个分区的采样数据。

4. 计算出数据量较大的分区通过RDD.sample进行重新抽样。

5. 通过抽样数组 candidates: ArrayBuffer[(K, wiegth)]计算出分区边界的数组BoundsArray

6. 在取数据时，如果分区数小于128则直接获取，如果大于128则通过二分法，获取当前Key属于那个区间，返回对应的BoundsArray下标即为partitionsID

# 1. **获取区间数组**

## 1.1. **给定样本总数**

给定总的数据抽样大小，最多1M的数据量(10^6)，最少20倍的RDD分区数量，也就是每个RDD分区至少抽取20条数据

```scala

class RangePartitioner(partitions,rdd) {

// 1. 计算样本大小
 val sampleSize =  math.min(20.0 * partitions, 1e6) 
}

```

## 1.2. **计算样本最大值**

 

RDD各分区中的数据量可能会出现倾斜的情况，乘于3的目的就是保证数据量小的分区能够采样到足够的数据，而对于数据量大的分区会进行第二次采样

 

```scala

class RangePartitioner(partitions,rdd) {
	

// 1. 计算样本大小
 val sampleSize =  math.min(20.0 * partitions, 1e6)
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(

		3.0 * sampleSize / rdd.partitions.length

	).toInt

}

 

```

## 1.3. **水塘抽样**

 

根据以上两个值进行水塘抽样，返回RDD的总数据量，分区ID和每个分区的采样数据。其中总数据量是估计值，不是通过rdd.count计算得到的

 

```scala

class RangePartitioner(partitions,rdd) {
// 1. 计算样本大小
 val sampleSize =  math.min(20.0 * partitions, 1e6)
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(

		3.0 * sampleSize / rdd.partitions.length

	).toInt

    
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = RangePartitioner.sketch(
    rdd.map(_._1), sampleSizePerPartition)

}
```

## 1.4. **是否需要二次采样**

 

如果有较大RDD存在，则按照平均值去采样的话数据量太少，容易造成数据倾斜，所以需要进行二次采样

 

判断是否需要重新采样方法：

样本数量占比乘以当前RDD的总行数大于预设的每个RDD最大抽取数量，说明这个RDD的数据量比较大，需要采样更多的数据：eg: 0.2*100=20<60;0.2*20000=2000>60

 

```scala

class RangePartitioner(partitions,rdd) {
// 1. 计算样本大小
 val sampleSize =  math.min(20.0 * partitions, 1e6)
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(

		3.0 * sampleSize / rdd.partitions.length

	).toInt  
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

## 1.5. **计算样本权重**

 

计算每个采样数据的权重占比，根据采样数据的ID和权重生成出RDD分区边界数组

 

权重计算方法：总数据量/当前RDD的采样数据量

 

```scala

class RangePartitioner(partitions,rdd) {
// 1. 计算样本大小
 val sampleSize =  math.min(20.0 * partitions, 1e6)
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(

		3.0 * sampleSize / rdd.partitions.length

	).toInt  
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = RangePartitioner.sketch(
    rdd.map(_._1), sampleSizePerPartition) 
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
 if (fraction * n > sampleSizePerPartition) {
	// 记录需要重新采样的RDD的ID
	imbalancedPartitions += idx

}else{

     
// 5. 计算样本权重
	val weight = (
	  // 采样数据的占比
		n.toDouble / sample.length).toFloat 
            for (key <- sample) {
			// 记录采样数据key和权重
              candidates += ((key, weight))
            }
	}
}

```

## 1.6. **二次抽样**

 

 对于数据分布不均衡的RDD分区，重新进行二次抽样。

二次抽样采用的是RDD的采样方法：RDD.sample

 

```scala

class RangePartitioner(partitions,rdd) {
// 1. 计算样本大小
 val sampleSize =  math.min(20.0 * partitions, 1e6)
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(

		3.0 * sampleSize / rdd.partitions.length

	).toInt  
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = RangePartitioner.sketch(
    rdd.map(_._1), sampleSizePerPartition) 
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
 if (fraction * n > sampleSizePerPartition) {
	// 记录需要重新采样的RDD的ID
	imbalancedPartitions += idx

}else{  
// 5. 计算样本权重
	val weight = (
	  // 采样数据的占比
		n.toDouble / sample.length).toFloat 
            for (key <- sample) {
			// 记录采样数据key和权重
              candidates += ((key, weight))
            }
	}

    
// 6. 对于数据分布不均衡的RDD分区，重新数据抽样
if (imbalancedPartitions.nonEmpty) {
	// 利用rdd的sample抽样函数API进行数据抽样
   val reSampled = imbalanced.sample(
    withReplacement = false, fraction, seed).collect()
}


```

## 1.7. **生成边界数组**

 

将最终的抽样数据计算出分区边界数组返回，边界数组里面存放的是RDD里面数据的key值，

比如最终返回的数组是：array[0,10,20,30..]

其中0,10,20,30是采样数据中的key值，对于每一条数据都会判断其在此数组的那个区间中间，例如有一条数据key值是3则其在0到10之间，属于第一个分区，同理Key值为15的数据在第二个分区

 

```scala

class RangePartitioner(partitions,rdd) {
// 1. 计算样本大小
 val sampleSize =  math.min(20.0 * partitions, 1e6)
// 2. 计算样本最大值
val sampleSizePerPartition = 
	math.ceil(

		3.0 * sampleSize / rdd.partitions.length

	).toInt  
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = RangePartitioner.sketch(
    rdd.map(_._1), sampleSizePerPartition) 
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
 if (fraction * n > sampleSizePerPartition) {
	// 记录需要重新采样的RDD的ID
	imbalancedPartitions += idx

}else{  
// 5. 计算样本权重
	val weight = (
	  // 采样数据的占比
		n.toDouble / sample.length).toFloat 
            for (key <- sample) {
			// 记录采样数据key和权重
              candidates += ((key, weight))
            }
	}
// 6. 对于数据分布不均衡的RDD分区，重新数据抽样
if (imbalancedPartitions.nonEmpty) {
	// 利用rdd的sample抽样函数API进行数据抽样
   val reSampled = imbalanced.sample(
    withReplacement = false, fraction, seed).collect()
}

    
// 7. 生成边界数组
RangePartitioner.determineBounds(candidates, partitions)

}

```

# 2. **水塘抽样算法**

 

水塘抽样概念：

它是一系列的随机算法，其目的在于从包含n个项目的集合S中选取k个样本，使得每条数据抽中的概率是k/n。其中n为一很大或未知的数量，尤其适用于不能把所有n个项目都存放到主内存的情况

 

我们可以：定义取出的行号为choice，第一次直接以第一行作为取出行 choice ，而后第二次以二分之一概率决定是否用第二行替换 choice ，第三次以三分之一的概率决定是否以第三行替换 choice ……，以此类推。由上面的分析我们可以得出结论，在取第n个数据的时候，我们生成一个0到1的随机数p，如果p小于1/n，保留第n个数。大于1/n，继续保留前面的数。直到数据流结束，返回此数，算法结束。

 详见：<https://www.iteblog.com/archives/1525.html>

# 3. **定位分区ID**

 

如果分区边界数组的大小小于或等于128的时候直接变量数组，否则采用二分查找法确定key属于某个分区。

 

## 3.1. **数组直接获取**

 

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

## 3.2. **二分法查找**

 

对于分区数大于128的情况，采样二分法查找

 ```scala

 // 根据RDD的key值返回对应的分区id。从0开始

  def getPartition(key: Any): Int = {

 // 如果分区数量大于128个，那么使用二分查找方法寻找对应k所属的下标;

      // 但是如果k在rangeBounds中没有出现，实质上返回的是一个负数(范围)或者是一个超过rangeBounds大小的数(最后一个分区，比所有数据都大)

      // Determine which binary search method to use only once.

      partition = binarySearch(rangeBounds, k)

      // binarySearch either returns the match location or -[insertion point]-1

      if (partition < 0) {

        partition = -partition-1

      }

      if (partition > rangeBounds.length) {

        partition = rangeBounds.length

      }

 ```



[查看完整源码...](https://my.oschina.net/freelili/blog/2987568/)