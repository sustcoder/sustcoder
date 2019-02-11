---
title: spark源码分析环境搭建
subtitle: run spark suite
description: 调试spark源码
keywords: [spark,源码,调试]
date: 2018-12-20
tags: [spark,源码]
category: [spark]
---

在学习spark的过程中发现很多博客对概念和原理的讲解存在矛盾或者理解不透彻，所以开始对照源码学习，发现根据概念总结去寻找对应源码，能更好理解，但随之而来的问题是好多源码看不懂，只跑example的话好多地方跑不到，但是结合测试类理解起来就方便多了。



forck一份源码，在未修改源码的情况下（修改源码后，比如加注释等，在编译阶段容易报错）,使用gitbash进入项目的根目录下，执行下面2条命令使用mvn进行编译：

1. 设置编译时mven内存大小

> export MAVEN_OPTS="-Xmx2g  -XX:ReservedCodeCacheSize=512m"

2.  `-T -4` :启动四个线程 , `- P`：将hadoop编译进去 

> ./build/mvn  -T 4 -Pyarn -Phadoop-2.6  -Dhadoop.version=2.6.0  -DskipTests clean package

3. 通过运行一个test测试是否编译完成

```scala
> build/sbt
> project core
> testOnly *RDDSuite -- -z "serialization"
```



**远程调试**

根据spark的[pmc建议](https://www.zhihu.com/question/24869894)使用远程调试运行test更方便，以下是使用远程调试步骤：

1. 使用idea通过maven导入导入项目：`File / Open，选定 Spark 根目录下的 pom.xml`,点击确定即可

2. 建立remote debugger

   1. 选取菜单项 Run > Edit Configurations... 点击左上角加号，选取 Remote 建立一套远程调试配置，并命名为“Remote Debugging (listen)”：
   2. 选择Debugger mode: `Listen to remote JVM`
   3. 选择Transport:`Socket`
   4. 配置完成后点击Debug，开始启动remote debugger的监听

   配置完成后如下图：

   ![1546052559018](https://sustblog.oss-cn-beijing.aliyuncs.com/blog%2F2018%2Fspark%2Fsrccode%2F1546052559018.png)

3. 在SBT中启动远程调试，打开idea中的`sbt shell`

   1. SBT 中的 settings key 是跟着当前 sub-project 走的，所以调试时需要切换到对应的模块，例如：切换到core模块，可以在sbt shell中执行以下命令

      > project core

   2. 设置监听模式远程调试选项：

      > set javaOptions in Test += "-agentlib:jdwp=transport=dt_socket,server=n,suspend=n,address=localhost:5005"

   3. 启动我们的 test case，[sbt test教程](http://www.scalatest.org/user_guide/using_the_runner)

      > testOnly *RDDSuite -- -z "basic operations"

   如果我们在test调用的代码里面打断点后，就可以进行调试了，结果如图：

   ![1546053639151](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/spark/srccode/1546053639151.png)



**报错信息**1

```scala
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 05:31 min
[INFO] Finished at: 2018-12-25T17:04:22+08:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-antrun-plugin:1.8:run (default) on project spark-core_2.11: An Ant BuildException has occured: Execute failed: java.io.IOException: Cannot run program "bash" (in directory "E:\data\gitee\fork-spark2.2.0\spark\core"): CreateProcess error=2, 系统找不到指定的文件。
[ERROR] around Ant part ...<exec executable="bash">... @ 4:27 in E:\data\gitee\fork-spark2.2.0\spark\core\target\antrun\build-main.xml
[ERROR] -> [Help 1]
[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
[ERROR]
[ERROR] After correcting the problems, you can resume the build with the command
[ERROR]   mvn <goals> -rf :spark-core_2.11
```

解决办法：Spark编译需要在bash环境下，直接在windows环境下编译会报不支持bash错误，

1. 利用git的bash窗口进行编译
2. 用`Windows Subsystem for Linux`,具体的操作，大家可以参考[这篇文章](https://medium.com/hungys-blog/windows-subsystem-for-linux-configuration-caf2f47d0dfb)



**报错信息2**

```scala
Output path E:\data\gitee\fork-spark2.2.0\spark\external\flume-assembly\target\scala-2.11\classes is shared between: Module 'spark-streaming-flume-assembly' production, Module 'spark-streaming-flume-assembly_2.11' production
Please configure separate output paths to proceed with the compilation.
TIP: you can use Project Artifacts to combine compiled classes if needed.
```

解决办法： 项目里面.idea文件夹下，有一个models.xml文件，里面同一个文件包含了两个不同的引用。猜测原因maven编译完成后，在idea中打开时又再次选择使用sbt进行编译，结果出现了两份导致，再删掉所有.iml文件后，重新通过mvn进行编译，并且在打开时不选择用sbt进行编译解决。



**报错信息**4

```scala
ERROR: Cannot load this JVM TI agent twice, check your java command line for duplicate jdwp options.
Disconnected from server
Error occurred during initialization of VM
agent library failed to init: jdwp
```

解决办法：执行了多次设置调试模式的方法`set javaOptions in Test += "-agentlib:jdwp=transport=dt_socket,server=n,suspend=n,address=localhost:5005"`导致，关掉sbt shell重启重试



**参考**：

https://spark.apache.org/developer-tools.html

https://www.zhihu.com/question/24869894 

