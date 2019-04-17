---
title: sparkStreaming翻译
subtitle: 官网文档翻译
description: sparkStreaming基本概念
keywords: [spark,streaming,入门,翻译]
author: liyz
date: 2018-11-16
tags: [sparkStreaming,翻译]
category: [翻译]
---

# Spark Streaming Programming Guide

## Overview 概述

Spark Streaming is an **extension**<sub>延伸</sub> of the core Spark API that enables **scalable**<sub>可扩展</sub>, **high-throughput**高吞吐, **fault-tolerant**<sub>容错</sub> stream processing of live data streams. Data can be **ingested** <sub>摄入获取</sub>from many sources like Kafka, Flume, Kinesis, or TCP sockets, and can be processed using **complex algorithms** <sub>复杂算法</sub>expressed with **high-level functions**<sub>高阶函数</sub> like `map`, `reduce`, `join` and `window`. Finally, processed data can be **pushed out to**<sub>输出到</sub> filesystems, databases, and live **dashboards**<sub>仪表盘</sub>. In fact, you can apply Spark’s machine learning and graph processing algorithms on data streams.

Spark Streaming 是 Spark Core API 的扩展, 它支持弹性的, 高吞吐的, 容错的实时数据流的处理. 数据可以通过多种数据源获取, 例如 Kafka, Flume, Kinesis 以及 TCP sockets, 也可以通过例如 `map`, `reduce`, `join`, `window` 等的高级函数组成的复杂算法处理. 最终, 处理后的数据可以输出到文件系统, 数据库以及实时仪表盘中. 事实上, 你还可以在 data streams（数据流）上使用 机器学习 以及 图计算算法.

![Spark Streaming](https://spark.apache.org/docs/latest/img/streaming-arch.png)

**Internally**<sub>在内部</sub>, it works as follows. Spark Streaming receives **live input data streams**<sub>实时输入流</sub> and **divides**<sub>切分</sub> the data into **batches**<sub>批此</sub>, which are then processed by the Spark **engine**<sub>引擎</sub> to generate the final stream of results in batches.

在内部, 它工作原理如下, Spark Streaming 接收实时输入数据流并将数据切分成多个 batch（批）数据, 然后由 Spark 引擎处理它们以生成最终的 stream of results in batches（分批流结果）.

![Spark Streaming](https://spark.apache.org/docs/latest/img/streaming-flow.png)

Spark Streaming provides a **high-level abstraction**<sub>高度抽象</sub> called **discretized**<sub>离散</sub> stream or *DStream*, which represents a **continuous**<sub>连续的</sub> stream of data. DStreams can be created either from input data streams from sources such as Kafka, Flume, and Kinesis, or by **applying**<sub>作用于</sub> high-level operations on other DStreams. Internally, a DStream **is represented as**代表 a sequence of RDDs

Spark Streaming 提供了一个名为 *discretized stream* 或 *DStream* 的高级抽象, 它代表一个连续的数据流. DStream 可以从数据源的输入数据流创建, 例如 Kafka, Flume 以及 Kinesis, 或者在其他 DStream 上进行高层次的操作以创建. 在内部, 一个 DStream 是通过一系列的 RDDs 来表示.

These underlying RDD transformations are computed by the Spark engine. The DStream operations hide most of these details and provide the developer with a higher-level API for convenience.

这些底层的 RDD 变换由 Spark 引擎（engine）计算。 DStream 操作隐藏了大多数这些细节并为了方便起见，提供给了开发者一个更高级别的 API 

## A Quick Example

Let’s say we want to count the number of words in text data received from a data server listening on a TCP socket. All you need to do is as follows.

我们想要计算从一个监听 TCP socket 的数据服务器接收到的文本数据（text data）中的字数. 你需要做的就是照着下面的步骤做

```scala
...
val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
val lines = ssc.socketTextStream("localhost", 9999)
val words = lines.flatMap(_.split(" "))
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
...
```

## Basic Concepts基本概念

To initialize a Spark Streaming program, a **StreamingContext** object has to be created which is the main **entry**<sub>入口</sub> **point of** <sub>要点</sub>all Spark Streaming functionality

为了初始化一个 Spark Streaming 程序, 一个 **StreamingContext** 对象必须要被创建出来，它是所有的 Spark Streaming 功能的主入口点

### **Initializing StreamingContext**

```scala
import org.apache.spark.streaming._
val sc = ... // existing SparkContext
val ssc = new StreamingContext(sc, Seconds(1))
```

After a context is defined, you have to do the following：

1. Define the **input sources**<sub>输入源</sub> by creating input DStreams.
2. Define the streaming **computations**<sub>计算</sub> by **applying**<sub>应用</sub> transformation and output operations to DStreams.
3. Start receiving data and processing it using `streamingContext.start()`.
4. Wait for the processing to be stopped (manually or due to any error) using `streamingContext.awaitTermination()`.
5. The processing can be **manually**<sub>手动的</sub> stopped using `streamingContext.stop()`.

在定义一个 context 之后,您必须执行以下操作：

1. 通过创建输入 DStreams 来定义输入源.
2. 通过应用转换和输出操作 DStreams 定义流计算（streaming computations）.
3. 开始接收输入并且使用 `streamingContext.start()` 来处理数据.
4. 使用 `streamingContext.awaitTermination()` 等待处理被终止（手动或者由于任何错误）.
5. 使用 `streamingContext.stop()` 来手动的停止处理.

##### Points to remember:

- Once a context has been started, no new streaming computations can be set up or added to it.
- Once a context has been stopped, it cannot be restarted.
- Only one StreamingContext can be active in a JVM at the same time.
- stop() on StreamingContext also stops the SparkContext. To stop only the StreamingContext, set the optional parameter of `stop()` called `stopSparkContext` to false.
- A SparkContext can be re-used to create multiple StreamingContexts, as long as the previous StreamingContext is stopped (without stopping the SparkContext) before the next StreamingContext is created.

**需要记住的几点:**

- 一旦一个 context 已经启动，将不会有新的数据流的计算可以被创建或者添加到它。.
- 一旦一个 context 已经停止，它不会被重新启动.
- 同一时间内在 JVM 中只有一个 StreamingContext 可以被激活.
- 在 StreamingContext 上的 stop() 同样也停止了 SparkContext 。为了只停止 StreamingContext ，设置 `stop()` 的可选参数，名叫 `stopSparkContext` 为 false.
- 一个 SparkContext 就可以被重用以创建多个 StreamingContexts，只要前一个 StreamingContext 在下一个StreamingContext 被创建之前停止（不停止 SparkContext）.

### Input DStreams and Receivers

Input DStreams are DStreams representing the stream of input data received from streaming sources. In the quick example, `lines` was an input DStream as it represented the stream of data received from the netcat server. Every input DStream (except file stream, discussed later in this section) is associated with a Receiver  object which receives the data from a source and stores it in Spark’s memory for processing.



<sub></sub>