# troubleshooting

**map和reduce执行效率不一致**

map端每输出一部分数据，reduce端就会进行拉去消费，并不等到map端全部执行完毕后，reduce端再去拉去数据执行，这样就会存在生产和消费效率不对等问题，如果map端每次输出的数据量很大，reduce端处理速度较慢。这个时候，再加上你的reduce端执行的聚合函数的代码，可能会创建大量的对象。就有可能OOM，所以需要合理设置Map和reduce端的Bufffer大小。

可通过如下参数设置shuffle read task的buffer缓冲大小

`spark.reducer.maxSizeInFlight`，默认值48m

**yarn-cluster模式下OOM**

有的时候，运行一些包含了spark sql的spark作业，可能会碰到yarn-client模式下，可以正常提交运行；yarn-cluster模式下，可能是无法提交运行的，会报出JVM的PermGen（永久代）的内存溢出，OOM。

yarn-client模式下，driver是运行在本地机器上的，spark使用的JVM的PermGen的配置，是本地的spark-class文件（spark客户端是默认有配置的），JVM的永久代的大小是128M，这个是没有问题的；但是呢，在yarn-cluster模式下，driver是运行在yarn集群的某个节点上的，使用的是没有经过配置的默认设置（PermGen永久代大小），82M。

所以，此时，如果对永久代的占用需求，超过了82M的话，但是呢又在128M以内；就会出现如上所述的问题，yarn-client模式下，默认是128M，这个还能运行；如果在yarn-cluster模式下，默认是82M，就有问题了。会报出PermGen Out of Memory error log。

既然是JVM的PermGen永久代内存溢出，那么就是内存不够用。咱们呢，就给yarn-cluster模式下的，driver的PermGen多设置一些。spark-submit脚本中，加入以下配置即可：
`--conf spark.driver.extraJavaOptions="-XX:PermSize=128M -XX:MaxPermSize=256M"`

**`chache的正确使用`**

```scala
// 报file not found的错误
usersRDD.cache()
usersRDD.count()
usersRDD.take()
// 正确姿势
usersRDD
usersRDD = usersRDD.cache()
val cachedUsersRDD = usersRDD.cache()
```



