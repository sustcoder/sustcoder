---
title: spark集群环境搭建
subtitle: spark on yarn
description: yarn环境搭建spark
keywords: [spark,yarn,环境,搭建]
date: 2018-09-28 15:27:04
tags: [spark,环境]
category: [spark]
---
# spark on yarn

# 软件安装

## 当前环境

hadoop环境搭建参考：[hadoop集群安装](https://my.oschina.net/freelili/blog/1834706)

- hadoop2.6
- spark-2.2.0-bin-hadoop2.6.tgz
- scala-2.11.12

## 安装scala

> tar -zxvf scala-2.11.12.tgz
>
> vi /etc/profile

添加以下内容

```shell
export SCALA_HOME=/home/hadoop/app/scala
export PATH=$PATH:$SCALA_HOME/bin
```

使配置生效

> source /etc/profile

查看scala版本号

> scala -version

注意： **用root账户修改完变量后，需要重新打开ssh链接，配置才能生效**

## 安装spark

> tar -zvf spark-2.2.0-bin-without-hadoop.tgz
>
> vi /etc/profile

添加以下内容

```shell
export SPARK_HOME=/home/hadoop/app/spark2.2.0
export PATH=$PATH:$SPARK_HOME/bin
```

修改spark环境变量

> cp spark-env.sh.template spark-env.sh
>
> vi conf/spark-evn.sh

添加以下内容

```shell
SPARK_DRIVER_MEMORY=512m
SPARK_DIST_CLASSPATH=$(/home/hadoop/app/hadoop-2.6.0/bin/hadoop classpath)
SPARK_LOCAL_DIRS=/home/hadoop/app/spark2.2.0
export SPARK_MASTER_IP=192.168.10.125

export JAVA_HOME=/home/app/jdk8
export SCALA_HOME=/home/hadoop/app/scala
export HADOOP_HOME=/home/hadoop/app/hadoop
export HADOOP_CONF_DIR=/home/hadoop/app/hadoop/etc/hadoop
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$SCALA_HOME/bin
```

使配置变量生效

> source /etc/profile

配置slaves

> vi slaves



```shell
node3
node4
```

将配置文件下发到从节点

```shell
scp slaves hadoop@node3:/home/hadoop/app/spark2.2.0/conf
scp slaves hadoop@node4:/home/hadoop/app/spark2.2.0/conf
scp spark-env.sh hadoop@node3:/home/hadoop/app/spark2.2.0/conf
scp spark-env.sh hadoop@node4:/home/hadoop/app/spark2.2.0/conf
```

## 启动

```shell
# 启动zookeeper,可能存在选举延迟，可多执行几次./zkServer.sh status查看启动结果
./runRemoteCmd.sh "/home/hadoop/app/zookeeper/bin/zkServer.sh start" zookeeper
# 在node2节点上执行,启动HDFS
sbin/start-dfs.sh
# 在node2节点上执行,启动YARN
sbin/start-yarn.sh 
# 在node4节点上面执行,启动resourcemanager
sbin/yarn-daemon.sh start resourcemanager
# 在node2上启动spark
sbin/start-all.sh
```

## 关闭

```shell
# 关闭spark
sbin/stop-all.sh
# 在node4上执行
sbin/yarn-daemon.sh stop resourcemanager
# 在node2上执行
sbin/stop-yarn.sh 
# 在node2上执行
sbin/stop-dfs.sh
# 关闭zookeeper
runRemoteCmd.sh "/home/hadoop/app/zookeeper/bin/zkServer.sh stop" zookeeper
```

查看启动情况

> jps

```shell
# hdfs进程
1661 NameNode
1934 SecondaryNameNode
1750 DataNode

# yarn进程
8395 ResourceManager
7725 NameNode

# namenode HA
8256 DFSZKFailoverController
7985 JournalNode

# zookeeper进程
1286 QuorumPeerMain

# spark进程
2551 Master
2641 Worker
```

## 管理界面

```http
hadoop:  http://node2:8088/
nameNode: http://node2:50070/
nodeManager:  http://node2:8042/
spark master:  http://node2:8080/
spark worker:  http://node2:8081/
spark jobs:  http://node2:4040/
```

## 运行示例

**Spark-shell**

```shell
vi test.text# 在文件中添加 hello spark
hdfs dfs -mkdir /test # 创建文件夹
hdfs dfs -put test.txt /test # 上传文件到hdfs
hdfs dfs -ls /test # 查看是否上传成功
./bin/spark-shell
sc.textFile("hdfs://node2:9000/test/test.txt") # 从hdfs上获取文件
sc.first() # 获取文件的第一行数据
```

**Run application locally(Local模式)**

> ./bin/spark-submit --class org.apache.spark.examples.SparkPi --master local[4] /home/hadoop/app/spark2.2.0/examples/jars/spark-examples_2.11-2.2.0.jar

**Run on a Spark standalone cluster(Standalone模式,使用Spark自带的简单集群管理器)**

> ./bin/spark-submit \
> --class org.apache.spark.examples.SparkPi \
> --master spark://node2:7077 \
> --executor-memory 512m \
> --total-executor-cores 4 \
> /home/hadoop/app/spark2.2.0/examples/jars/spark-examples_2.11-2.2.0.jar 10

**Run on yarn(YARN模式，使用YARN作为集群管理器)**

>./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode client examples/jars/spark-examples*.jar 10


# 注意事项

> HADOOP_CONF_DIR=/home/hadoop/app/hadoop/etc/hadoop

确保 `HADOOP_CONF_DIR` 或者 `YARN_CONF_DIR` 指向包含 Hadoop 集群的（客户端）配置文件的目录。这些配置被用于写入 HDFS 并连接到 YARN ResourceManager 。此目录中包含的配置将被分发到 YARN 集群，以便 application（应用程序）使用的所有的所有 containers（容器）都使用相同的配置。如果配置引用了 Java 系统属性或者未由 YARN 管理的环境变量，则还应在 Spark 应用程序的配置（driver（驱动程序），executors（执行器），和在客户端模式下运行时的 AM ）。



>  SPARK_DIST_CLASSPATH=$(/home/hadoop/app/hadoop-2.6.0/bin/hadoop classpath)

Pre-build with user-provided Hadoop: 属于“Hadoop free”版,不包含hadoop的jar等，这样，下载到的Spark，可应用到任意Hadoop 版本。但是需要在spark的spar-evn.sh中指定配置hadoop的安装路径。



> spark-submit

如果用户的应用程序被打包好了，它可以使用 `bin/spark-submit` 脚本来启动。这个脚本负责设置 Spark 和它的依赖的 classpath，并且可以支持 Spark 所支持的不同的 Cluster Manager 以及 deploy mode（部署模式）:

```shell
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
```

一些常用的 options（选项）有 :

- `--class`: 您的应用程序的入口点（例如。 `org.apache.spark.examples.SparkPi`)
- `--master`: standalone模式下是集群的 master URL，on yarn模式下值是`yarn`
- `--deploy-mode`: 是在 worker 节点(`cluster`) 上还是在本地作为一个外部的客户端(`client`) 部署您的 driver(默认: `client`)
- `--conf`: 按照 key=value 格式任意的 Spark 配置属性。对于包含空格的 value（值）使用引号包 “key=value” 起来。
- `application-jar`: 包括您的应用以及所有依赖的一个打包的 Jar 的路径。该 URL 在您的集群上必须是全局可见的，例如，一个 `hdfs://` path 或者一个 `file://` 在所有节点是可见的。
- `application-arguments`: 传递到您的 main class 的 main 方法的参数，如果有的话。

# 异常处理

**执行脚本**

> ./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode client /home/hadoop/app/spark2.2.0/examples/jars/spark-examples_2.11-2.2.0.jar

**报错信息一**

```shell
Application application_1537990303043_0001 failed 2 times due to AM Container for appattempt_1537990303043_0001_000002 exited with  exitCode: -103
Diagnostics: 
Container [pid=2344,containerID=container_1537990303043_0001_02_000001] is running beyond virtual memory limits. 
Current usage: 74.0 MB of 1 GB physical memory used; 
2.2 GB of 2.1 GB virtual memory used. Killing container.
```

**问题原因**

虚拟机物理内存设置的是1G，则对应虚拟内存最大为1*2.1=2.1GB,实际使用了2.2[此处疑问：为什么就使用了2.2，单个任务默认分配1024M，加上一个任务的Container默认1024M导致吗？]，所以需要扩大虚拟内存的比例，或者限制container和task的大小，或者关闭掉对虚拟内存的检测。

**解决方法**

修改`yarn-site.xml`文件，新增以下内容，详情原因请参考：[YARN 内存参数详解](https://sustcoder.github.io/2018/09/27/YARN%20%E5%86%85%E5%AD%98%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3/)

```xml
  <property>
      <name>yarn.nodemanager.vmem-pmem-ratio</name>
      <value>3</value>
      <description>虚拟内存和物理内存比率，默认为2.1</description>
  </property>
  <property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
    <description>不检查虚拟内存，默认为true</description>
  </property>
```

**报错二**

```java
Exception in thread "main" org.apache.spark.SparkException: Yarn application has already ended! It might have been killed or unable to launch application master.
	at org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend.waitForApplication(YarnClientSchedulerBackend.scala:85)
	at org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend.start(YarnClientSchedulerBackend.scala:62)
	at org.apache.spark.scheduler.TaskSchedulerImpl.start(TaskSchedulerImpl.scala:173)
	at org.apache.spark.SparkContext.<init>(SparkContext.scala:509)
	at org.apache.spark.SparkContext$.getOrCreate(SparkContext.scala:2509)
	at org.apache.spark.sql.SparkSession$Builder$$anonfun$6.apply(SparkSession.scala:909)
	at org.apache.spark.sql.SparkSession$Builder$$anonfun$6.apply(SparkSession.scala:901)
	at scala.Option.getOrElse(Option.scala:121)
	at org.apache.spark.sql.SparkSession$Builder.getOrCreate(SparkSession.scala:901)
	at org.apache.spark.examples.SparkPi$.main(SparkPi.scala:31)
	at org.apache.spark.examples.SparkPi.main(SparkPi.scala)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:755)
	at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:180)
	at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:205)
	at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:119)
	at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
18/09/04 17:01:43 INFO util.ShutdownHookManager: Shutdown hook called

```

**问题原因**

以上报错是在伪集群上运行时报错信息，具体报错原因未知，在切换到真正的集群环境后无此报错

# 配置链接

**hadoop**

- [core-site.xml](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/hadoop/core-site.xml)

- [hdfs-site.xml](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/hadoop/hdfs-site.xml)

- [mapred-site.xml](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/hadoop/mapred-site.xml)

- [yarn-site.xml](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/hadoop/yarn-site.xml)

- [hadoop-env.sh](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/hadoop/hadoop-env.sh)

- [masters](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/hadoop/masters)

- [slaves](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/hadoop/slaves)

**zookeeper**

- [zoo.cfg](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/zookeeper/zoo.cfg)

**spark**

- [spark-env.sh](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/spark/spark-env.sh)
- [slaves](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018//hadoop/conf/spark/slaves)

**env**

- [hosts](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/env/hosts)
- [profile](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/env/profile)

**tools**

- [deploy.conf](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/tools/deploy.conf)
- [deploy.sh](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/tools/deploy.sh)
- [runRemoteCmd.sh](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/hadoop/conf/tools/runRemoteCmd.sh)