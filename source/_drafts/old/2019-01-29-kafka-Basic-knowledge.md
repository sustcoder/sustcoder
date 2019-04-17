---
title: kafka 概览 
subtitle: kafka基础
description: kafka概念
keywords: [kafka,概念]
date: 2019-01-29
tags: [kafka,概念]
category: [kafka]
---

## 与传统的mq区别

### 吞吐量

高吞吐是kafka需要实现的核心目标之一，为此kafka在消息写入，消息保存和消息发送的三个阶段均做了优化：

1. 消息写入阶段

- 数据批量发送
- 数据压缩

2. 消息保存阶段

- 数据磁盘持久化：**直接使用linux 文件系统的cache**，来高效缓存数据。
- zero-copy：减少IO操作步骤，传统的数据发送需要发送4次上下文切换，**采用sendfile**系统调用之后，数据直接在内核态交换，系统上下文切换减少为2次

3. 消息发送阶段

- Topic划分为多个partition，提高并行度

### 负载均衡

负载均衡主要包括消息的分割及消息的备份以及broker和consumer的动态加入和离开。

- producer根据用户指定的算法，将消息发送到指定的partition
- 存在多个partiiton，每个partition有自己的replica，每个replica分布在不同的Broker节点上
- 多个partition需要选取出lead partition，lead partition负责读写，并由zookeeper负责fail over
- 通过zookeeper管理broker与consumer的动态加入与离开

### 拉取系统

由于kafka broker会持久化数据，broker没有内存压力，因此，consumer非常适合采取pull的方式消费数据，具有以下几点好处：

- 简化kafka设计
- consumer根据消费能力自主控制消息拉取速度
- consumer根据自身情况自主选择消费模式，例如批量，重复消费，从尾端开始消费等

### 可扩展性

当需要增加broker结点时，新增的broker会向zookeeper注册，而producer及consumer会根据注册在zookeeper上的watcher感知这些变化，并及时作出调整。

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka1.png)

## kafka名词解释

- Producer ：消息生产者，就是向kafka broker发消息的客户端。
- Consumer ：消息消费者，向kafka broker取消息的客户端
- Topic ：topic可以理解为数据库的一张表或者文件系统里面的一个目录。
- Consumer Group （CG）：这是kafka用来实现一个topic消息的广播（发给所有的consumer）和单播（发给任意一个consumer）的手段。一个partition只能被GC里面的一个消费者消费，不同消费者组可以消费同一个partition。
- Broker ：一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。
- Partition：一个topic分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。kafka只保证按一个partition中的顺序将消息发给consumer，不保证一个topic的整体（多个partition间）的顺序。
-  Offset：kafka的存储文件都是按照offset.kafka来命名，用offset做名字的好处是方便查找。例如你想找位于2049的位置，只要找到2048.kafka的文件即可。

### Broker Leader的选举

Kakfa Broker集群受Zookeeper管理。所有的Kafka Broker节点一起去Zookeeper上注册一个临时节点，因为只有一个Kafka Broker会注册成功，其他的都会失败，所以这个成功在Zookeeper上注册临时节点的这个Kafka Broker会成为**Kafka Broker Controller**，其他的Kafka broker叫Kafka Broker follower。

**Kafka动态维护了一个同步状态的副本的集合（a set of in-sync replicas），简称ISR**（除了controller之外其他broker节点信息），在这个集合中的节点都是和leader保持高度一致的，任何一条消息必须被这个集合中的每个节点读取并追加到日志中了，才回通知外部这个消息已经被提交了。因此这个集合中的任何一个节点随时都可以被选为leader.ISR在ZooKeeper中维护。ISR中有f+1个节点，就可以允许在f个节点down掉的情况下不会丢失消息并正常提供服。ISR的成员是动态的，如果一个节点被淘汰了，当它重新达到“同步中”的状态时，他可以重新加入ISR.这种leader的选择方式是非常快速的，适合kafka的应用场景。

### Consumer

 Consumer处理partition里面的message的时候是o（1）顺序读取的。所以必须维护着上一次读到哪里的offsite信息。high level API,offset存于Zookeeper中，low level API的offset由自己维护。

### Consumergroup

各个consumer（consumer 线程）可以组成一个组（Consumer group ），partition中的每个message只能被组（Consumer group ）中的一个consumer（consumer 线程）消费，如果一个message可以被多个consumer（consumer 线程）消费的话，那么这些consumer必须在不同的组

如果想多个不同的业务都需要这个topic的数据，起多个consumer group就好了，大家都是顺序的读取message，offsite的值互不影响。这样没有锁竞争，充分发挥了横向的扩展性，吞吐量极高。这也就形成了分布式消费的概念

新启动的consumer默认从partition队列最头端最新的地方开始阻塞的读message

最优的设计就是，consumer group下的consumer thread的数量等于partition数量，一个consumer thread处理一个partition，这样效率是最高的。

如果producer的流量增大，当前的topic的parition数量=consumer数量，这时候的应对方式就是很想扩展：增加topic下的partition，同时增加这个consumer group下的consumer。

### Consumer Rebalance

触发条件：消费者数量或者broker数量发生变化时

- Consumer增加或删除会触发 Consumer Group的Rebalance
- Broker的增加或者减少都会触发 Consumer Rebalance

### Topic & Partition

Topic相当于传统消息系统MQ中的一个队列queue，producer端发送的message必须指定是发送到哪个topic，但是**不需要指定topic下的哪个partition**，因为kafka会把收到的message进行load balance，均匀的分布在这个topic下的不同的partition上（ hash(message) % [broker数量]  ）。物理上存储上，这个topic会分成一个或多个partition，**每个partiton相当于是一个子queue**。在物理结构上，每个partition对应一个物理的目录（文件夹），文件夹命名是[topicname]_[partition]_[序号]，一个topic可以有无数多的partition，根据业务需求和数据量来设置。在kafka配置文件中可随时更高num.partitions参数来配置更改topic的partition数量，在创建Topic时通过参数指定parittion数量。Topic创建之后通过Kafka提供的工具也可以修改partiton数量。

   一般来说，（1）**一个Topic的Partition数量大于等于Broker的数量，可以提高吞吐率**。（2）同一个Partition的Replica尽量分散到不同的机器，高可用。

### Partition&Replica

**每个partition可以在其他的kafka broker节点上存副本**，默认的副本数为1，可通过`default.replication.factor`配置副本数，以便某个kafka broker节点宕机不会影响这个kafka集群。存replica副本的方式是按照kafka broker的顺序存。例如有5个kafka broker节点，某个topic有3个partition，每个partition存2个副本，那么partition1存broker1,broker2，partition2存broker2,broker3。。。以此类推（**replica副本数目不能大于kafka broker节点的数目**，否则报错。这里的replica数其实就是partition的副本总数，其中包括一个leader，其他的就是copy副本）。这样如果某个broker宕机，其实整个kafka内数据依然是完整的。但是，replica副本数越高，系统虽然越稳定，但是回来带资源和性能上的下降；replica副本少的话，也会造成系统丢数据的风险。

1. 怎样传送消息：producer先把message发送到partition leader，再由leader发送给其他partition follower。

2. 在向Producer发送ACK前需要保证有多少个Replica已经收到该消息：根据ack配的个数而定

3. 怎样处理某个Replica不工作的情况：如果这个部工作的partition replica不在ack列表中，就是producer在发送消息到partition leader上，partition leader向partition follower发送message没有响应而已，这个不会影响整个系统，也不会有什么问题。如果这个不工作的partition replica在ack列表中的话，producer发送的message的时候会等待这个不工作的partition replca写message成功，但是会等到time out，然后返回失败因为某个ack列表中的partition replica没有响应，此时**kafka会自动的把这个部工作的partition replica从ack列表中移除**，以后的producer发送message的时候就不会有这个ack列表下的这个部工作的partition replica了。 

4. 怎样处理Failed Replica恢复回来的情况：如果这个partition replica之前不在ack列表中，那么启动后重新受Zookeeper管理即可，之后producer发送message的时候，partition leader会继续发送message到这个partition follower上。如**果这个partition replica之前在ack列表中，此时重启后，需要把这个partition replica再手动加到ack列表中**。（ack列表是手动添加的，出现某个部工作的partition replica的时候自动从ack列表中移除的）

### Partition leader&follower

**partition也有leader和follower之分**。leader是主partition，producer写kafka的时候先写partition leader，再由partition leader push给其他的partition follower。**partition leader与follower的信息受Zookeeper控制**，一旦partition leader所在的broker节点宕机，zookeeper会冲其他的broker的partition follower上选择follower变为parition leader。

### Broker

- **Broker没有副本机制**，一旦broker宕机，该broker的消息将都不可用。
- Broker不保存订阅者的状态，由订阅者自己保存
- Broker的无状态特点导致其需要定期删除数据以及当订阅者故障时需要从最小offset重新消费，也就是不能够保证exactly once

![](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka6.png)


## Consumer与topic关系

- 每个group中可以有多个consumer，每个consumer属于一个consumer group； 

- 对于Topic中的一条特定的消息，只会被订阅此Topic的每个group中的其中一个consumer消费

- kafka只能保证一个partition中的消息被某个consumer消费时是顺序的；事实上，从Topic角度来说,当有多个partitions时,消息仍不是全局有序的。

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka2.png)

## Kafka消息的分发

Producer客户端负责消息的分发 

- kafka集群中的任何一个broker都可以向producer提供metadata信息,这些metadata中包含”集群中存活的servers列表”/”partitions leader列表”等信息； 

- 当producer获取到metadata信息之后, producer将会和Topic下所有partition leader保持socket连接； 

-  消息由producer直接通过socket发送到broker，中间不会经过任何”路由层”，事实上，消息被路由到哪个partition上由producer客户端决定； 

比如可以采用”random”“key-hash”“轮询”等,如果一个topic中有多个partitions,那么在producer端实现”消息均衡分发”是必要的。 

- 在producer端的配置文件中,开发者可以指定partition路由的方式。

![message](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/message.png)

## Producer消息发送的应答机制 

  设置发送数据是否需要服务端的反馈,有三个值0,1,-1 

  0: producer不会等待broker发送ack 

  1: 当leader接收到消息之后发送ack 

  -1: 当所有的follower都同步消息成功后发送ack 



## kafka中的zookeeper

### kafka在zookeeper中存储路径

```:negative_squared_cross_mark:
/broker/ids/[0...N]  ## broker node注册
/broker/topics/[topic]/partitions/[0...N]  ## topic 和 partitions
/consumers/[group_id]/ids/[consumer_id] ## consumer注册
/consumers/[group_id]/offsets/[topic]/[partition_id] ## 每个consumer group目前所消费的partition中最大的offset
/consumers/[group_id]/owners/[topic]/[partition_id] ## 表明parttion的消费者
```

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka4.png)

[查看更多](https://blog.csdn.net/ouyang111222/article/details/51094912)

### 控制器与zookeeper关系

在老版本的kafka中没有控制器，都是由各个broker直接和zookeeper通信，这样造成资源浪费和zookeeper脑裂和羊群效应的发生，所以在新的版本中Kafka集群中会有一个或者多个broker，其中有一个broker会被选举为控制器（Kafka Controller），controller和zoookeeper进行通信，监听partition,topic,broker等变化并更新zookeeper上的节点信息。

控制器在竞选成功后会初始化一个上下文，并将上下文信息同步给其他broker节点。对于新增事件，统一维护在线程安全的linkedblockingQueue中，然后按照FIFO的原则顺序的处理这些事件，维护多线程的安全。

![](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka8.png)

[查看更多](https://blog.csdn.net/u013256816/article/details/80865540)

## 消息可靠性

消息投递可靠性：

一个消息如何算投递成功，Kafka提供了三种模式：

1.  ack=0:第一种是啥都不管，发送出去就当作成功，这种情况当然不能保证消息成功投递到broker；

2. ack=-1:第二种是Master-Slave模型，只有当Master和所有Slave都接收到消息时，才算投递成功，这种模型提供了最高的投递可靠性，但是损伤了性能；

3. ack=1:第三种模型，即只要Master确认收到消息就算投递成功；实际使用时，根据应用特性选择，绝大多数情况下都会中和可靠性和性能选择第三种模型

消息在broker上的可靠性，因为消息会持久化到磁盘上，所以如果正常stop一个broker，其上的数据不会丢失；但是**如果broker不正常stop，可能会使存在页面缓存来不及写入磁盘的消息丢失，这可以通过配置flush页面缓存的周期、阈值缓解，但是同样会频繁的写磁盘会影响性能**，又是一个选择题，根据实际情况配置。

消息消费的可靠性：

**Kafka提供的是“At least once”模型**，因为消息的读取进度由offset提供，offset可以由消费者自己维护也可以维护在zookeeper里，但是当消息消费后consumer挂掉，offset没有即时写回，就有可能发生重复读的情况。

## 文件存储机制

kafka文件存储优点：

- 数据磁盘持久化：**直接使用linux 文件系统的cache**，来高效缓存数据。
- zero-copy：减少IO操作步骤，传统的数据发送需要发送4次上下文切换，**采用sendfile**系统调用之后，数据直接在内核态交换，系统上下文切换减少为2次
- 数据批量发送
- 数据压缩
- Topic划分为多个partition，提高并行度

### 从IO角度看kafka数据传输

实际场景中用户调整page cache的手段并不太多，更多的还是通过**管理好broker端的IO来间接影响page cache从而实现高吞吐量**。

理想流程：

producer发送消息给broker—>broker将数据直接写入page cache中—>fllower broker从page cache中拉取数据—>consumer从page cache中拉取数据。

所以如果想提高IO的效率，就应该在数据未从page cache中删除之前去获取，从而避免走磁盘访问。具体有以下场景可能需要访问磁盘：

- consumer消费速度过慢，在拉取数据时数据已经写入broker磁盘（具体写入时间由操作系统决定）

- 老版本consumer,由于老版本consumer的消息格式和现有不同需要走JVM导致整个过程缓慢
- 日志压缩，broker在定期做日志压缩时会消耗掉一定的IO和内存

page cache的调优

- 设置合理(主要是偏小)的Java Heap size，Kafka对于JVM堆的需求并不是特别大，6~10GB大小的JVM堆是一个比较合理的数值
- 调节内核的文件预取(prefetch)：文件预取是指将数据从磁盘读取到page cache中，防止出现缺页中断(page fault)而阻塞。


### topic中partition存储分布

假设实验环境中Kafka集群只有一个broker，xxx/message-folder为数据文件存储根目录，在Kafka broker中server.properties文件配置(参数log.dirs=xxx/message-folder)，

例如创建2个topic名 称分别为report_push、launch_info, partitions数量都为partitions=4,存储路径和目录规则为：

```java
xxx/message-folder  // 数据文件存储根目录
  |--report_push-0  // topic report_push partition 0 目录
  |--report_push-1
  |--report_push-2
  |--report_push-3
  |--launch_info-0
  |--launch_info-1
  |--launch_info-2
  |--launch_info-3
```

在Kafka文件存储中，同一个topic下有多个不同partition，每个partition为一个目录，**partiton命名规则为topic名称+有序序号**，第一个partiton序号从0开始，序号最大值为partitions数量减1。**消息发送时都被发送到一个topic，其本质就是一个目录**。

### partiton中文件存储方式

- **每个partion(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中**。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。
- 每个partiton只需要支持顺序读写就行了，segment文件生命周期由服务端配置参数决定。

### partiton中segment文件结构

每个part在内存中对应一个index，记录每个segment中的第一条消息偏移。

- **segment file组成：由2大部分组成，分别为index file和data file**，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件.
- segment文件命名规则：partion全局的第一个segment从0开始，后续**每个segment文件名为上一个全局partion的最大offset(偏移message数)**。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

![kafka5](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka5.png)

segment中index<—->data file对应关系物理结构:

index : message序号,物理偏移地址

log: 

![kafka5](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka7.png)



### 如何通过offset查找message

例如读取offset=368776的message，需要通过下面2个步骤查找。

- **第一步通过二分法查找所在segment file**

  上述图2为例，其中00000000000000000000.index表示最开始的文件，起始偏移量(offset)为0.**第二个文件 00000000000000368769.index的消息量起始偏移量为368770 = 368769 + 1**.同样，第三个文件00000000000000737337.index的起始偏移量为737338=737337 + 1，其他后续文件依次类推，以起始偏移量命名并排序这些文件，只要根据offset **二分查找**文件列表，就可以快速定位到具体文件。

  当offset=368776时定位到00000000000000368769.index|log

- **再定位到的segnent index中顺序查找**

  当offset=368776时，依次定位到00000000000000368769.index的元数据物理位置和 00000000000000368769.log的物理偏移地址，**然后再通过00000000000000368769.log顺序查找直到 offset=368776为止**。

segment index file采取**稀疏索引存储**方式，它减少索引文件大小，通过mmap可以直接内存操作，稀疏索引为数据文件的每个对应message设置一个元数据指针,它 比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间。

### 消息格式

对于日志来说，一条记录以"\n"结尾，或者通过其它特定的分隔符分隔，这样就可以从文件中拆分出一条一条的记录，不过这种格式更适用于文本，对于Kafka来说，需要的是二进制的格式。所以，Kafka使用了另一种经典的格式：在消息前面固定长度的几个字节记录下这条消息的大小(以byte记)，所以Kafka的记录格式变成了：

`Offset MessageSize Message`

消息被以这样格式append到文件里，在读的时候通过MessageSize可以确定一条消息的边界。

**MessageSet**

MessageSet是由多条记录组成的，而不是消息，这就决定了一个MessageSet实际上不需要借助其它信息就可以从它对应的字节流中切分出消息，而这决定了更重要的性质：**Kafka的压缩是以MessageSet为单位的**。将MessageSet压缩后作为另一个Message的value。Kafka的消息是可以递归包含的，也就是前边"value"字段的说明“Kafka supports recursive messages in which case this may itself contain a message set"。但是注意：**Kafka中的一个Message最多只含有一个MessageSe**否则会报：MessageSizeTooLargeException

具体地说，对于Kafka来说，可以对一个MessageSet做为整体压缩，把压缩后得到的字节数组作为一条Message的value。于是，Message既可以表示未压缩的单条消息，也可以表示压缩后的MessageSet。

从0.8.x版本开始到现在的1.1.x版本，Kafka的消息格式也经历了3个版本。

**v0版本**
对于Kafka消息格式的第一个版本，我们把它称之为v0，在Kafka 0.10.0版本之前都是采用的这个消息格式。注意如无特殊说明，我们只讨论消息未压缩的情形。 

![img](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/kafka/kafka3.png)


上左图中的“RECORD”部分就是v0版本的消息格式，大多数人会把左图中的整体，即包括offset和message size字段都都看成是消息，因为每个Record（v0和v1版）必定对应一个offset和message size。每条消息都一个offset用来标志它在partition中的偏移量，这个offset是逻辑值，而非实际物理偏移值，message size表示消息的大小，这两者的一起被称之为日志头部（LOG_OVERHEAD），固定为12B。LOG_OVERHEAD和RECORD一起用来描述一条消息。与消息对应的还有消息集的概念，消息集中包含一条或者多条消息，消息集不仅是存储于磁盘以及在网络上传输（Produce & Fetch）的基本形式，而且是kafka中压缩的基本单元，详细结构参考上图。

下面来具体陈述一下消息（Record）格式中的各个字段，从crc32开始算起，各个字段的解释如下：

- crc32（4B）：crc32校验值。校验范围为magic至value之间。
- magic（1B）：消息格式版本号，此版本的magic值为0。
- attributes（1B）：消息的属性。总共占1个字节，低3位表示压缩类型：0表示NONE、1表示`GZIP`、2表示`SNAPPY`、3表示`LZ4`（LZ4自Kafka 0.9.x引入），其余位保留。
- key length（4B）：表示消息的key的长度。如果为-1，则表示没有设置key，即key=null。
- key：可选，如果没有key则无此字段。
- value length（4B）：实际消息体的长度。如果为-1，则表示消息为空。
- value：消息体。可以为空，比如tomnstone消息。

v0版本中一个消息的最小长度（RECORD_OVERHEAD_V0）为crc32 + magic + attributes + key length + value length = 4B + 1B + 1B + 4B + 4B =14B，**也就是说v0版本中一条消息的最小长度为14B，如果小于这个值，那么这就是一条破损的消息而不被接受**。

**v1版本**

kafka从0.10.0版本开始到0.11.0版本之前所使用的消息格式版本为v1，其比v0版本就多了一个timestamp字段，表示消息的时间戳。

加时间戳的原因：在没有时间戳的情况下，移动replica后，replica的创建时间会时当前时间，会影响到log segment的删除策略，另外假如时间戳可以更好的支持流处理。

v1版本的magic字段值为1。v1版本的attributes字段中的低3位和v0版本的一样，还是表示压缩类型，而第4个bit也被利用了起来：0表示timestamp类型为CreateTime，而1表示tImestamp类型为LogAppendTime，其他位保留。v1版本的最小消息（RECORD_OVERHEAD_V1）大小要比v0版本的要大8个字节，即22B。

**v2版本**

kafka从0.11.0版本开始所使用的消息格式版本为v2，这个版本的消息相比于v0和v1的版本而言改动很大，同时还参考了Protocol Buffer而引入了变长整型（Varints）和ZigZag编码。Varints是使用一个或多个字节来序列化整数的一种方法，数值越小，其所占用的字节数就越少。ZigZag编码以一种锯齿形（zig-zags）的方式来回穿梭于正负整数之间，以使得带符号整数映射为无符号整数，这样可以使得绝对值较小的负数仍然享有较小的Varints编码值，比如-1编码为1,1编码为2，-2编码为3。


## 配置文件server.properties

- `broker.id=0` :  #当前机器在集群中的唯一标识，和zookeeper的myid性质一样

- `port=19092`  #当前kafka对外提供服务的端口默认是9092

- `host.name=192.168.7.100`  #这个参数默认是关闭的，在0.8.1有个bug，DNS解析问题，失败率的问题。

- `num.network.threads=3`  #这个是borker进行网络处理的线程数

- `num.io.threads=8` #这个是borker进行I/O处理的线程数

- `log.dirs=/opt/kafka/kafkalogs/ ` #消息存放的目录，可以配置多个目录，以“,”分割，上面的num.io.threads要大于这个目录的个数这个目录，如果配置多个目录，新创建的topic他把消息持久化的地方是，当前以逗号分割的目录中，那个分区数最少就放那一个

- `socket.send.buffer.bytes=102400` #发送缓冲区buffer大小，数据不是一下子就发送的，先回存储到缓冲区了到达一定的大小后在发送，能提高性能

- `socket.receive.buffer.bytes=102400` #kafka接收缓冲区大小，当数据到达一定大小后在序列化到磁盘

- `socket.request.max.bytes=104857600` #这个参数是向kafka请求消息或者向kafka发送消息的请请求的最大数，这个值不能超过java的堆栈大小

- `num.partitions=1` #默认的分区数，一个topic默认1个分区数

- `log.retention.hours=168` #默认消息的最大持久化时间，168小时，7天

- `message.max.byte=5242880`  #消息保存的最大值5M

- `default.replication.factor=2`  #kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务

- `replica.fetch.max.bytes=5242880`  #取消息的最大直接数

- `log.segment.bytes=1073741824` #这个参数是：因为kafka的消息是以追加的形式落地到文件，当超过这个值的时候，kafka会新起一个文件

- `log.retention.check.interval.ms=300000` #每隔300000毫秒去检查上面配置的log失效时间（log.retention.hours=168 ），到目录查看是否有过期的消息如果有，删除

- `log.cleaner.enable=false` #是否启用log压缩，一般不用启用，启用的话可以提高性能

- `zookeeper.connect=192.168.7.100:12181,192.168.7.101:12181,192.168.7.107:1218` #设置zookeeper的连接端口

## 常用命令

1、启动kafka：   `./bin/kafka-server-start.sh ./config/server.properties`

2、创建主题：` ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test`

3、查看主题： `./bin/kafka-topics.sh --list --zookeeper 127.0.0.1:2181`

4、删除主题：`./bin/kafka-topics.sh --zookeeper 127.0.0.1:2181 --delete --topic test `（默认不进行删除，只是打上了删除标记。设置server.properties文件内“delete.topic.enable=true”，并且重启Kafka就可以了。）

5、发送消息：`./bin/kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic test`

6、接收消息：`./bin/kafka-console-consumer.sh --zookeeper 127.0.0.1:2181 --topic test --from-beginning`

## kafka优化思路

主要从4个方面考虑优化：吞吐量(throughput)、延时(latency)、持久性(durability)和可用性(availability)。“鱼和熊掌不可兼得”——你没有办法最大化所有目标。这4者之间必然存在着权衡(tradeoff)。常见的tradeoff包括：吞吐量和延时权衡、持久性和可用性之间权衡。但是当我们考虑整个系统时通常都不能孤立地只考虑其中的某一个方面，而是需要全盘考量。

### partition

- 使用随机分区
- 增加partition

### producer

producer端可通过配置batch大小，缓存大小，是否压缩和重试次数等来提高吞吐量

- batch.size = 100000 - 200000，默认是16384，通常都太小了
- compression.type = lz4，如果是低延迟则不压缩
- acks = 1，如果是持久性，则配置为all
- retries = 0，如果是持久性则配置相对较大的值，比如5 ~ 10默认为3
- buffer.memory：如果分区数很多则适当增加 (默认是32MB)

### consumer

- fetch.min.bytes如果想达到高吞吐则配置为`10-100`，如果是低延迟则配置为1，默认为1

- Consumer 的套接字缓冲区(socket buffers)，以应对数据的高速流入,或者设置为-1，让底层根据网络情况自动设置
- Consumers 应当使用固定大小的缓冲区，而且最好是使用堆外内存(off-heap)，固定大小的缓冲区能够阻止 Consumer 将过多的数据拉到堆栈上，以至于 JVM 花费掉其所有的时间去执行垃圾回收，进而无法履行其处理消息的本质工作。
- 在 JVM 上运行各种 Consumers 时，请警惕垃圾回收对它们可能产生的影响，长时间垃圾回收的停滞，可能导致 ZooKeeper 的会话被丢弃、或 Consumer group 处于再均衡状态。对于 Broker 来说也如此，如果垃圾回收停滞的时间太长，则会产生集群掉线的风险。

### broker

- num.replica.fetchers：如果发生ISR频繁进出的情况或follower无法追上leader的情况则适当增加该值，但通常不要超过CPU核数+1
- num.recovery.threads.per.data.dir = log.dirs中配置的目录数

[查看更多](https://www.cnblogs.com/huxi2b/p/6936348.html)


## 问题导读

##### 如何安全的再均衡及如何避免不必要的再均衡？

当分区被重新分配给另一个消费者时，消费者当前读取的状态丢失，它有可能需要去刷新缓存，在他恢复状态之前会拖慢应用程序。

##### 数据丢失的集中可能性？

1. 生产者数据写入时丢失

- 提交策略不当
- 提交偏移量不正确
- 关闭了不完全首领选举，acks=all,但是生产者写入数据时，首领挂了，正在重新选举中，kafka会返回首领不可用的响应，但是生产者没有正确处理。

2. 接收数据时的数据丢失

- 副本还未同步，但是使用了首领接收即确认后首领宕机
- 使用了不完全首领选举

3. 运维层的数据丢失

- 网络故障

##### 数据压缩格式都有哪几种

怎样知道发送的数据有没有丢失，压缩的是那些内容？

kafka默认是以二进制的形式组织message的，同时也可以设置为字符串，json等格式的数据。

```scala
// 默认是二进制数组形式：        
props.put("serializer.class", "kafka.serializer.DefaultEncoder");
// 发送的数据是String
props.put("serializer.class", "kafka.serializer.StringEncoder")  
```

压缩算法：NONE、GZIP、SNAPPY、LZ4。Apache Kafka 2.1.0正式支持ZStandard，优点是压缩比高

通过crc32校验值在broker接收消息和consumer消费消息时针对每一条Message可通过crc32进行校验。

kafka压缩的是message的value，但是可以将一个messageSet放到Messsage中压缩以提高压缩比率。

##### 当key为null时划分到那个区

我们往Kafka发送消息时一般都是将消息封装到KeyedMessage类中：

```scala
val message = new KeyedMessage[String, String](topic, key, content)
producer.send(message)
```
Kafka会根据传进来的key进行hash计算其分区ID。但是这个Key可以不传，根据Kafka的官方文档描述：如果key为null，那么Producer将会把这条消息发送给随机的一个Partition。

具体的随机方式：

0.8版本中：在key为null的情况下，Kafka并不是每条消息都随机选择一个Partition；而是每隔`topic.metadata.refresh.interval.ms`才会随机选择一次，使用这种伪随机的以此来减少服务器端的sockets数。

##### kafka2.0的新功能

- 更全面的数据安全支持，KIP-255 里面添加了一个框架，我们可以使用OAuth2 bearer tokens 来对访问 [Kafka](https://www.iteblog.com/archives/tag/kafka/)Brokers 进行权限控制。
- 保证在线升级的方便性，在这一次的 2.0.0 版本中，更多相关的属性被加了进来，详情请参见 KIP-268、KIP-279、KIP-283 等等
- 进一步加强了 Kafka 的可监控性，包括添加了很多系统静态属性以及动态健康指标，请参见 KIP-223、KIP-237、KIP-272 等等。

##### KSQL

ksql是kafka提供的一个sql引擎，但是它和我们传统意义上的sql不一样，传统的sql是建立在静态的表之上的，而ksql是建立在流数据（日志）之上的，更适合于基于事件的统计分析，例如日志报警机制，在多长时间内发送了多少erro日志等，以及联机数据整合，将多个数据源通过KSQL整合成一个输出。

[查看详情](https://www.iteblog.com/archives/2254.html)

##### topic如何放到不同的broker上

- 副本因子不能大于 Broker 的个数；
- 第一个分区（编号为0）的第一个副本放置位置是随机从 `brokerList` 选择的；第一个放置的分区副本一般都是 Leader，其余的都是 Follower 副本。
- 其他分区的第一个副本放置位置相对于第0个分区依次往后移。也就是如果我们有5个 Broker，5个分区，假设第一个分区放在第四个 Broker 上，那么第二个分区将会放在第五个 Broker 上；第三个分区将会放在第一个 Broker 上；第四个分区将会放在第二个 Broker 上，依次类推；
- 剩余的副本相对于第一个副本放置位置其实是随机产生的；

[查看更多](https://www.iteblog.com/archives/2219.html)

##### 新建的分区会在哪个目录下创建

在启动 [Kafka](https://www.iteblog.com/archives/tag/kafka/) 集群之前，我们需要配置好 `log.dirs` 参数，其值是 [Kafka](https://www.iteblog.com/archives/tag/kafka/) 数据的存放目录，这个参数可以配置多个目录，目录之间使用逗号分隔，通常这些目录是分布在不同的磁盘上用于提高读写性能。

如果 `log.dirs` 参数配置了多个目录，**Kafka 会在含有分区目录最少的文件夹中创建新的分区目录**，分区目录名为 Topic名+分区ID。注意，是分区文件夹总数最少的目录，而不是磁盘使用量最少的目录！也就是说，如果你给 `log.dirs` 参数新增了一个新的磁盘，新的分区目录肯定是先在这个新的磁盘上创建直到这个新的磁盘目录拥有的分区目录不是最少为止。

这种实现没有考虑到磁盘容量的负载，以及新加入的磁盘读写的负载。

[查看更多](https://www.iteblog.com/archives/2231.html)

##### 客户端是如何找到leader分区的

 Kafka 是使用 Scala 语言编写的，但是其支持很多语言的客户端，包括：C/C++、PHP、Go以及Ruby等等，他们怎么和kafka通讯呢，这是因为 Kafka 内部实现了一套基于TCP层的协议，只要使用这种协议与Kafka进行通信，就可以使用很多语言来操作Kafka。

其中负责如何找到Leader分区是由Metadata协议完成的。Metadata包含了kafka包含哪些主题，每个主题的分区以及分区的leaer所在broker的地址和端口等信息。

客户端只需要构造一个 `TopicMetadataRequest`，Kafka 将会把内部所有的主题相关的信息发送给客户端。

```scala
// metadata包含信息
partitionId=0
    leader=Some(id:1,host:1.iteblog.com,port:9092) 
    isr=Vector(id:1,host:1.iteblog.com,port:9092)
    replicas=Vector(id:1,host:1.iteblog.com,port:9092, 	
                    id:8,host:8.iteblog.com,port:9092)
```

[查看更多](https://www.iteblog.com/archives/2215.html)

##### broker是无状态的吗

从消费者角度来看，broker并不记录offset的偏移量，从整个角度来说是无状态的。

从metadata角度来看，每个broker都维护了相同的一份metadata cache,在partition数量，broker的新增或者删除时会更新这个cache，以保证producer端无论请求那个broker都能够获得metadata信息。

[查看更多](https://www.cnblogs.com/huxi2b/p/8440429.html)

##### kafka分区分配策略

每个分区只能由同一个消费组内的一个consumer来消费。那么问题来了，同一个 Consumer Group 里面的 Consumer 是如何知道该消费哪些分区里面的数据呢？

在 Kafka 内部存在两种默认的分区分配策略：Range 和 RoundRobin。具体策略可通过参数配置：`partition.assignment.strategy`。

当以下事件发生时，Kafka 将会进行一次分区分配：

- 同一个 Consumer Group 内新增消费者
- 消费者离开当前所属的Consumer Group，包括shuts down 或 crashes
- 订阅的主题新增分区

**Range策略**

range分区的对象是topic，会对同一个topic里面的partition进行排序，然后除以consumer数量，得到的步长就是每个consumer消费的数量，对consumer也进行了排序，每个consumer消费对应步长里面的partition。如果不能均分，则那么前几个消费者线程将多消费一个分区。这样的缺点也就是考前的消费者压力较大。例如：

```scala
topic A 的分区数量是10，则排序后是：0,1,2,3,4,5,6,7,8,9
consumer有两个C1和C2,C1有一个线程C1-0,C2有两个线程C2-0,C2-1
则对应的消费情况是：
C1-0 将消费 0, 1, 2, 3 分区（10&3=1分配给c1-0消费）
C2-0 将消费 4, 5, 6 分区
C2-1 将消费 7, 8, 9 分区
```

##### RoundRobin 策略

使用RoundRobin策略有两个前提条件必须满足：

- 同一个Consumer Group里面的所有消费者的num.streams必须相等；
- 每个消费者订阅的主题必须相同。

所以这里假设前面提到的2个消费者的num.streams = 2。RoundRobin策略的工作原理：将所有主题的分区组成 TopicAndPartition 列表，然后对 TopicAndPartition 列表按照 hashCode（`data.hashCode ``%` `numPartitions`） 进行排序,遍历消费者线程给分配。

```scala
假如按照 hashCode 排序完的topic-partitions组依次为T1-5, T1-3, T1-0, T1-8, T1-2, T1-1, T1-4, T1-7, T1-6, T1-9，我们的消费者线程排序为C1-0, C1-1, C2-0, C2-1，最后分区分配的结果为：

C1-0 将消费 T1-5, T1-2, T1-6 分区；
C1-1 将消费 T1-3, T1-1, T1-9 分区；
C2-0 将消费 T1-0, T1-4 分区；
C2-1 将消费 T1-8, T1-7 分区；
```

##### 如何选择合适的topics和partitions的数量

经过测试，在producer端，单个partition的吞吐量通常是在10MB/s左右。在consumer端，单个partition的吞吐量依赖于consumer端每个消息的应用逻辑处理速度。因此，我们需要对consumer端的吞吐量进行测量。

**一开始，我们可以基于当前的业务吞吐量为kafka集群分配较小的broker数量，随着时间的推移，我们可以向集群中增加更多的broker**，然后在线方式将适当比例的partition转移到新增加的broker中去。通过这样的方法，我们可以在满足各种应用场景（包括基于key消息的场景）的情况下，保持业务吞吐量的扩展性。

通常情况下，kafka集群中越多的partition会带来越高的吞吐量。但是，我们必须意识到集群的partition总量过大或者单个broker节点partition过多，都会对系统的可用性和消息延迟带来潜在的影响。

增加过多的分区带来的负面影响

- 越多的分区需要打开更多地文件句柄，一个分区有index和data两个句柄。
- 更多地分区会导致更高的不可用性，每个broker和controller的恢复都有耗时，这个耗时由量变到质变。
- 越多的分区可能增加端对端的延迟。一个broker上所有的repartition操作只有一个线程。
- 越多的partition意味着需要客户端需要更多的内存。

[查看详情](https://www.iteblog.com/archives/2209.html)

##### 集群扩展与重新分区

添加集群操作：

我们往已经部署好的集群里面添加机器是最正常不过的需求，而且添加起来非常地方便，我们需要做的事是从已经部署好的节点中复制相应的配置文件，然后把里面的broker id修改成全局唯一的，最后启动这个节点即可将它加入到现有集群中。

但是问题来了，**新添加的Kafka节点并不会自动地分配数据，所以无法分担集群的负载，除非我们新建一个topic**。但是现在我们想手动将部分分区移到新添加的Kafka节点上，Kafka内部提供了相关的工具来重新分布某个topic的分区。

重新分区：

Kafka自带的`kafka-reassign-partitions.sh`工具来重新分布分区。总共三步：

1. 生成分区计划。generate模式，给定需要重新分配的Topic，自动生成reassign plan
2. 执行分区计划。execute模式，根据指定的reassign plan重新分配Partition
3. 查看分区结果是否正确。verify模式，验证重新分配Partition是否成功z

[查看详情](https://www.iteblog.com/archives/1611.html)

##### producer如何动态感知topic分区数变化

如果在Kafka Producer往Kafka的Broker发送消息的时候用户通过命令修改了改主题的分区数，Kafka Producer能动态感知吗？答案是可以的。那是立刻就感知吗？不是，是过一定的时间(`topic.metadata.refresh.interval.ms`参数决定)才知道分区数改变的。

每隔指定的时间，客户端会主动去更新topicPartitionInfo（`HashMap[String, TopicMetadata]`）信息。

在启动Kafka Producer往Kafka的Broker发送消息的时候，用户修改了该Topic的分区数，Producer可以在最多`topic.metadata.refresh.interval.ms`的时间之后感知到，此感知同时适用于`async`和`sync`模式，并且可以将数据发送到新添加的分区中。

##### 可以人为设置分区数和复制因子吗

我们无法通过Producer相关的API设定分区数和复制因子的，因为Producer相关API创建topic的是通过读取`server.properties`文件中的`num.partitions`和`default.replication.factor`的。

我们可以通过Kafka提供的`AdminUtils.createTopic`函数或者`TopicCommand`来创建topic

```scala
def createTopic(zkClient: ZkClient, 
      topic: String,
      partitions: Int,   
      replicationFactor: Int,  
      topicConfig: Properties = new Properties)

val arguments = Array("--create", "--zookeeper", zk, "--replication-factor", "2", "--partition", "2", "--topic", "iteblog")
TopicCommand.main(arguments)
```

[概念查看详情](https://my.oschina.net/hblt147/blog/2992936)

##### 日志留存策略

日志留存方式相关的策略类型主要有删除和压缩两种：`delete`和`compact`

删除策略又分为三种：删除的单位是一个分区下面的一个日志段，即LogSegment，而且当前正在写入的日志，无论哪种策略都不会被删除。

- 基于空间的维度，默认-1，不启动，可设置broker级别或者topic级别
- 基于时间的维度，默认为保存7天，根据传入的时间戳和服务器时间比较
- 基于起始位移的维度，适用于流处理场景，在同一个partition下，删除掉指定offset之前的日志。

[查看详情](https://www.cnblogs.com/huxi2b/p/8042099.html)

##### consumer动态修改topic订阅

定义一个线程安全的存放topic的集合对象`ConcurrentLinkedQueue`，每次消费时判断此集合是否有需要监听的topic，如果有则调用`consumer.subscribe(topics)`更新consumer的订阅。

```scala
public static void main(String[] args) {
	// 存储需要变化的topic    
	final ConcurrentLinkedQueue<String> subscribedTopics
    	= new ConcurrentLinkedQueue<>();
		// 最开始的订阅列表：atopic、btopic
        consumer.subscribe(Arrays.asList("atopic", "btopic"));
        while (true) {
            //表示每2秒consumer就有机会去轮询一下订阅状态是否需要变更，也可以在此间隔执行一些其他相关的操作，比如定期日志记录等
            consumer.poll(2000); 
            // 本例不关注消息消费，因此每次只是打印订阅结果！
            System.out.println(consumer.subscription());
            if (!subscribedTopics.isEmpty()) {
                Iterator<String> iter = subscribedTopics.iterator();
                List<String> topics = new ArrayList<>();
                while (iter.hasNext()) {
                    topics.add(iter.next());
                }
                subscribedTopics.clear();
                consumer.subscribe(topics); // 重新订阅topic
            }
        }   
       // 创建另一个测试线程，启动后首先暂停10秒然后变更topic订阅
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    // swallow it.
                }
                // 变更为订阅topic： btopic， ctopic
                subscribedTopics.addAll(Arrays.asList("btopic", "ctopic"));
            }
        };
        new Thread(runnable).start();
```

##### topic的管理

Kafka官方提供了两个脚本来管理topic，包括topic的增删改查。其中`kafka-topics.sh`负责topic的创建与删除；`kafka-configs.sh`脚本负责topic的修改和查询，但很多用户都更加倾向于使用程序API的方式对topic进行操作。

**创建topic**:`AdminUtils.createTopic`

```java
ZkUtils zkUtils = ZkUtils.apply("localhost:2181", 30000, 30000, JaasUtils.isZkSecurityEnabled());
// 创建一个单分区单副本名为t1的topic
AdminUtils.createTopic(zkUtils, "t1", 1, 1, new Properties(), RackAwareMode.Enforced$.MODULE$);
zkUtils.close();
```

**删除topic**:`AdminUtils.deleteTopic(zkUtils, "t1")`

```java
ZkUtils zkUtils = ZkUtils.apply("localhost:2181", 30000, 30000, JaasUtils.isZkSecurityEnabled());
// 删除topic 't1'
AdminUtils.deleteTopic(zkUtils, "t1");
zkUtils.close();
```

比较遗憾地是，**不管是创建topic还是删除topic，目前Kafka实现的方式都是后台异步操作的**，而且没有提供任何回调机制或返回任何结果给用户，**所以用户除了捕获异常以及查询topic状态之外似乎并没有特别好的办法可以检测操作是否成功**。

**查询topic**:`AdminUtils.fetchEntityConfig`

```java
ZkUtils zkUtils = ZkUtils.apply("localhost:2181", 30000, 30000, JaasUtils.isZkSecurityEnabled());
// 获取topic 'test'的topic属性属性
Properties props = AdminUtils.fetchEntityConfig(zkUtils, ConfigType.Topic(), "test");
// 查询topic-level属性
Iterator it = props.entrySet().iterator();
while(it.hasNext()){
    Map.Entry entry=(Map.Entry)it.next();
    Object key = entry.getKey();
    Object value = entry.getValue();
    System.out.println(key + " = " + value);
}
zkUtils.close();
```

**修改topic**:`AdminUtils.changeTopicConfig`

```java
ZkUtils zkUtils = ZkUtils.apply("localhost:2181", 30000, 30000, JaasUtils.isZkSecurityEnabled());
Properties props = AdminUtils.fetchEntityConfig(zkUtils, ConfigType.Topic(), "test");
// 增加topic级别属性
props.put("min.cleanable.dirty.ratio", "0.3");
// 删除topic级别属性
props.remove("max.message.bytes");
// 修改topic 'test'的属性
AdminUtils.changeTopicConfig(zkUtils, "test", props);
zkUtils.close();
```
