---
title: spark源码解析之RangePartitioner源码
subtitle: spark rdd 分区器
description: ,RangePartitioner详解
keywords: [水塘抽样,RangePartitioner,分区]
date: 2018-12-10
tags: [spark,源码解析]
category: [spark]
---

  

## 分区过程概览

RangePartitioner分区执行原理：

 计算总体的数据抽样大小sampleSize，计算规则是：至少每个分区抽取20个数据或者最多1M的数据量。

1. 根据sampleSize和分区数量计算每个分区的数据抽样样本数量最大值sampleSizePrePartition
2. 根据以上两个值进行水塘抽样，返回RDD的总数据量，分区ID和每个分区的采样数据。
3. 计算出数据量较大的分区通过RDD.sample进行重新抽样。
4. 通过抽样数组 candidates: ArrayBuffer[(K, wiegth)]计算出分区边界的数组BoundsArray
5. 在取数据时，如果分区数小于128则直接获取，如果大于128则通过二分法，获取当前Key属于那个区间，返回对应的BoundsArray下标即为partitionsID

[前往分步解析...]((https://sustcoder.github.io/2018/12/10/sparkCore-sourceCodeAnalysis_partitioner/))

完整代码：

**RangePartitioner**

```scala
class RangePartitioner(partitions,rdd) {
// 1. 计算样本大小
 val sampleSize = math.min(20.0 * partitions, 1e6)
// 2. 计算样本最大值
val sampleSizePerPartition = math.ceil(3.0 * sampleSize / rdd.partitions.length).toInt
// 3. 进行抽样，返回总数据量，分区ID和样本数据
val (numItems, sketched) = RangePartitioner.sketch(
    rdd.map(_._1), sampleSizePerPartition)
// 4. 是否需要二次采样
val imbalancedPartitions = 	mutable.Set.empty[Int]
//  5. 保存样本数据的集合buffer:包含数据和权重
val candidates = ArrayBuffer.empty[(K, Float)]
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

## rangeBounds

```scala
 // An array of upper bounds for the first (partitions - 1) partitions
  private var rangeBounds: Array[K] = {
    if (partitions <= 1) {
      Array.empty
    } else {
      //  This is the sample size we need to have roughly balanced output partitions, capped at 1M.
      //  给定总的数据抽样大小，最多1M的数据量(10^6)，最少20倍的RDD分区数量，也就是每个RDD分区至少抽取20条数据
      val sampleSize = math.min(20.0 * partitions, 1e6)
      // Assume the input partitions are roughly balanced and over-sample a little bit.
      // RDD各分区中的数据量可能会出现倾斜的情况，乘于3的目的就是保证数据量小的分区能够采样到足够的数据，而对于数据量大的分区会进行第二次采样
      val sampleSizePerPartition = math.ceil(3.0 * sampleSize / rdd.partitions.length).toInt
      // 从rdd中抽样得到的数据，返回值:(总数据量， Array[分区id，当前分区的数据量，当前分区抽取的数据])
      val (numItems, sketched) = RangePartitioner.sketch(rdd.map(_._1), sampleSizePerPartition)
      if (numItems == 0L) {
        // 如果总的数据量为0(RDD为空)，那么直接返回一个空的数组
        Array.empty
      } else {
        // If a partition contains much more than the average number of items, we re-sample from it
        // to ensure that enough items are collected from that partition.
        // 计算是否需要重新采样：如果分区包含的数据量远远大于平均采样的数据量则重新进行分区
        // 样本占比：计算总样本数量和总记录数的占比，占比最大为1.0
        val fraction = math.min(sampleSize / math.max(numItems, 1L), 1.0)
        //  保存样本数据的集合buffer:包含数据和权重
        val candidates = ArrayBuffer.empty[(K, Float)]
        // 保存数据分布不均衡的分区id(数据量超过fraction比率的分区)
        val imbalancedPartitions = mutable.Set.empty[Int]
        // 遍历抽样数据
        sketched.foreach { case (idx, n, sample) =>
          if (fraction * n > sampleSizePerPartition) {
            //  样本数量占比乘以当前RDD的总行数大于预设的每个RDD最大抽取数量，说明这个RDD的数据量比较大，需要采样更多的数据：eg: 0.2*100=20<60;0.2*20000=2000>60
            // 如果样本占比乘以当前分区中的数据量大于之前计算的每个分区的抽象数据大小，那么表示当前分区抽取的数据太少了，该分区数据分布不均衡，需要重新抽取
            imbalancedPartitions += idx // 记录需要重新采样的RDD的ID
          } else {
            // The weight is 1 over the sampling probability.
            val weight = (n.toDouble / sample.length).toFloat // 采样数据的占比，RDD越大，权重越大
            for (key <- sample) {
              candidates += ((key, weight))
            }
          }
        }
        // 对于数据分布不均衡的RDD分区，重新进行数据抽样
        if (imbalancedPartitions.nonEmpty) {
          // Re-sample imbalanced partitions with the desired sampling probability.
          val imbalanced = new PartitionPruningRDD(rdd.map(_._1), imbalancedPartitions.contains)
          val seed = byteswap32(-rdd.id - 1)
          // 利用rdd的sample抽样函数API进行数据抽样
          val reSampled = imbalanced.sample(withReplacement = false, fraction, seed).collect()
          val weight = (1.0 / fraction).toFloat
          candidates ++= reSampled.map(x => (x, weight))
        }
        // 将最终的抽样数据计算出分区边界数组返回，边界数组里面存放的是RDD里面数据的key值，
        // 比如array[0,10,20,30..]表明：key值在0到10的在第一个RDD，key值在10到20的在第二个RDD
        RangePartitioner.determineBounds(candidates, partitions)
      }
    }
  }
```

## **sketch**

```scala
  def sketch[K : ClassTag](
      rdd: RDD[K],
      sampleSizePerPartition: Int): (Long, Array[(Int, Long, Array[K])]) = {
    val shift = rdd.id
    // val classTagK = classTag[K] // to avoid serializing the entire partitioner object
    val sketched = rdd.mapPartitionsWithIndex { (idx, iter) =>
      val seed = byteswap32(idx ^ (shift << 16))
      /*水塘抽样：返回抽样数据和RDD的总数据量*/
      val (sample, n) = SamplingUtils.reservoirSampleAndCount(
        iter, sampleSizePerPartition, seed)
      Iterator((idx, n, sample))
    }.collect()
    // 计算所有RDD的总数据量
    val numItems = sketched.map(_._2).sum
    (numItems, sketched)
  }
```

## determineBounds

```scala
 /** 依据候选中的权重划分分区，权重值可以理解为该Key值所代表的元素数目 返回一个数组，长度为partitions - 1,第i个元素作为第i个分区内元素key值的上界
   *  Determines the bounds for range partitioning from candidates with weights indicating how many
   *  items each represents. Usually this is 1 over the probability used to sample this candidate.
   *
   * @param candidates unordered candidates with weights 抽样数据，包含了每个样本的权重
   * @param partitions number of partitions 分区数量
   * @return selected bounds
   */
  def determineBounds[K : Ordering : ClassTag](
      candidates: ArrayBuffer[(K, Float)],
      partitions: Int): Array[K] = {
    val ordering = implicitly[Ordering[K]]
    //依据Key进行排序，升序，所以按区间分区后，各个分区是有序的
    val ordered = candidates.sortBy(_._1)
    // 采样数据总数
    val numCandidates = ordered.size
    // //计算出权重和
    val sumWeights = ordered.map(_._2.toDouble).sum
    // 计算出步长：权重总数相当于预计数据总量，除以分区数就是每个分区的数量，得到的值即是按区间分割的区间步长
    val step = sumWeights / partitions
    var cumWeight = 0.0
    // 初始化target值为区间大小
    var target = step
    val bounds = ArrayBuffer.empty[K]
    var i = 0
    var j = 0
    var previousBound = Option.empty[K]
    // 遍历采样数据
    while ((i < numCandidates) && (j < partitions - 1)) {
      val (key, weight) = ordered(i)
      // 计算采样数据在当前RDD中的位置，如果大于区间大小则：记录边界KEY值
      cumWeight += weight
      if (cumWeight >= target) {
        // Skip duplicate values. // 相同key值处于相同的Partition中，key值不同可以进行分割
        if (previousBound.isEmpty || ordering.gt(key, previousBound.get)) {
          bounds += key //记录边界
          target += step
          j += 1
          previousBound = Some(key)
        }
      }
      i += 1
    }
    bounds.toArray
  }
```

**getPartition**

```scala
// 根据RDD的key值返回对应的分区id。从0开始
  def getPartition(key: Any): Int = {
    // 强制转换key类型为RDD中原本的数据类型
    val k = key.asInstanceOf[K]
    var partition = 0
    if (rangeBounds.length <= 128) {
      // If we have less than 128 partitions naive search
      // 如果分区数据小于等于128个，那么直接本地循环寻找当前k所属的分区下标
      // ordering.gt(x,y):如果x>y,则返回true
      while (partition < rangeBounds.length && ordering.gt(k, rangeBounds(partition))) {
        partition += 1
      }
    } else {
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
    }
    //  根据数据排序是升序还是降序进行数据的排列，默认为升序
    if (ascending) {
      partition
    } else {
      rangeBounds.length - partition
    }
  }
```

