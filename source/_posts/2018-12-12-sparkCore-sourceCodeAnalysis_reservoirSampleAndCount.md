---
title: RangePartitioner实现算法reservoirSampleAndCount
subtitle: 水塘抽样算法实现
description: sparkCore算法之reservoir
keywords: [spark,reservoir,水塘抽样]
date: 2018-12-12
tags: [spark,算法]
category: [spark]
---

## 简介

reservoir的作用是：**在不知道文件总行数的情况下，如何从文件中随机的抽取一行？**即是说如果最后发现文字档共有N行，则每一行被抽取的概率均为1/N？

我们可以：定义取出的行号为choice，第一次直接以第一行作为取出行 choice ，而后第二次以二分之一概率决定是否用第二行替换 choice ，第三次以三分之一的概率决定是否以第三行替换 choice ……，以此类推。由上面的分析我们可以得出结论，**在取第n个数据的时候，我们生成一个0到1的随机数p，如果p小于1/n，保留第n个数。大于1/n，继续保留前面的数。直到数据流结束，返回此数，算法结束。**

这个问题的扩展就是：如何从未知或者很大样本空间随机地取k个数？亦即是说，如果档案共有N ≥ k行，则每一行被抽取的概率为k/N。

　　根据上面（随机取出一元素）的分析，我们可以把上面的1/n变为k/n即可。思路为：**在取第n个数据的时候，我们生成一个0到1的随机数p，如果p小于k/n，替换池中任意一个为第n个数。大于k/n，继续保留前面的数。直到数据流结束，返回此k个数。但是为了保证计算机计算分数额准确性，一般是生成一个0到n的随机数，跟k相比，道理是一样的。**

## **伪代码**

```
从S中抽取首k项放入「水塘」中
对于每一个S[j]项（j ≥ k）：
   随机产生一个范围0到j的整数r
   若 r < k 则把水塘中的第r项换成S[j]项
```

```scala
/*
  S has items to sample, R will contain the result
*/
ReservoirSample(S[1..n], R[1..k])
  // fill the reservoir array
  for i = 1 to k
      R[i] := S[i]
 
  // replace elements with gradually decreasing probability
  for i = k+1 to n
    j := random(1, i)   // important: inclusive range
    if j <= k
        R[j] := S[i]
```

## 实现概述

1. 获取到需要抽样RDD分区的样本大小k和分区的所有KEY数组input
2. 初始化抽样结果集reservoir为分区前K个KEY值
3. 如果分区的总数小于预计样本大小k,则将当前分区的所有数据作为样本数据，否则到第四步
4. 遍历分区里所有Key组成的数组input
5. 生成随机需要替换input数组的下标，如果下标小于K则替换
6. 返回抽取的key值数组和当前分区的总数据量： (reservoir, l)

## 实现源码

```scala
/**
   * Reservoir sampling implementation that also returns the input size.
   *
   * @param input:RDD的分区里面的key组成的Iterator
   * @param k :抽样大小=
   		val sampleSize = math.min(20.0 * partitions, 1e6)
   		val k=math.ceil(3.0 * sampleSize / rdd.partitions.length).toInt
   * @param seed random seed:选取随机数的种子
   * @return (samples, input size)
   */
  def reservoirSampleAndCount[T: ClassTag](
      input: Iterator[T],
      k: Int,
      seed: Long = Random.nextLong())
    : (Array[T], Long) = {
    val reservoir = new Array[T](k)
    // Put the first k elements in the reservoir.
    // 初始化水塘数据为input的钱K个数，即：reservoir数组中放了RDD分区的前K个key值
    var i = 0
    while (i < k && input.hasNext) {
      val item = input.next()
      reservoir(i) = item
      i += 1
    }

    // If we have consumed all the elements, return them. Otherwise do the replacement.
    // 如果当前的RDD总数小于预设值的采样量则全部作为采样数据并结束采样
    if (i < k) {
      // If input size < k, trim the array to return only an array of input size.
      val trimReservoir = new Array[T](i)
      System.arraycopy(reservoir, 0, trimReservoir, 0, i)
      (trimReservoir, i)
    } else {
      // If input size > k, continue the sampling process.
      var l = i.toLong
      val rand = new XORShiftRandom(seed)
      // 遍历所有的key
      while (input.hasNext) {
        val item = input.next()
        l += 1
        // There are k elements in the reservoir, and the l-th element has been
        // consumed. It should be chosen with probability k/l. The expression
        // below is a random long chosen uniformly from [0,l)
        // 计算出需要替换的数组下标
        // 选取第n个数的概率是：n/l; 如果随机替换数组值的概率是p=rand.nextDouble，
        // 则如果p<k/l;则替换池中任意一个数，即： p*l < k 则进行替换，用p*l作为随机替换的下标
        val replacementIndex = (rand.nextDouble() * l).toLong
        if (replacementIndex < k) {
          // 替换reservoir[随机抽取的下标]的值为input[l]的值item
          reservoir(replacementIndex.toInt) = item
        }
      }
      (reservoir, l)
    }
  }
```

参考：https://www.iteblog.com/archives/1525.html