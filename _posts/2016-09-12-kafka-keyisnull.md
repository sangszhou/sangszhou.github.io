---
layout: post
title:  "Kafka walk through"
date:   "2016-09-12 00:00:00"
categories: kafka
keywords: kafka
---

## 使用命令行运行 Producer, Consumer

### 使用命令行创建 Producer 并发送消息

bin/kafka-producer.sh --create producer 

> This is a message

> another message 



## Producer

### 使用代码发送消息

```scala
  def createProducer[K, V](brokerList: String, serializerClass: String, requiredAcks: String, 
    partitionerClass: String = "kafka.producer.DefaultPartitioner"): Producer[K, V] = {
    
    val props = new Properties()
    props.put("metadata.broker.list", brokerList)
    props.put("serializer.class", serializerClass)
    props.put("request.required.acks", requiredAcks)
    props.put("partitioner.class", partitionerClass)
    val config = new ProducerConfig(props)
    new Producer[K, V](config)
  }
  
val kafkaProducer: Producer[String, String] = KafkaUtils.createProducer(broker, serializer, acks)

// def send(messages : java.util.List[kafka.producer.KeyedMessage[K, V]])
kafkaProducer.send(contentList.asJava)
```


## Consumer

### 使用 scala 代码

```scala
def createConsumers(zookeeperConnect: String, consumerGroup: String, sessionTimeout: String, 
    syncTime: String, commitInterval: String, connectionTimeout: String, consumerTimeout: String, 
    autoOffsetReset: String, topic: String, numOfConsumers: Int): Array[KafkaStream[Array[Byte], Array[Byte]]] = {
  
  val props: Properties = new Properties
  props.put("zookeeper.connect", zookeeperConnect)
  props.put("group.id", consumerGroup)
  props.put("zookeeper.session.timeout.ms", sessionTimeout)
  props.put("zookeeper.sync.time.ms", syncTime)
  props.put("auto.commit.interval.ms", commitInterval)
  props.put("zookeeper.connection.timeout.ms", connectionTimeout)
  props.put("consumer.timeout.ms", consumerTimeout)
  props.put("auto.offset.reset", autoOffsetReset)
  
  val config = new ConsumerConfig(props)
  val consumerConnector = kafka.consumer.Consumer.createJavaConsumerConnector(config)
  val topicCountMap: util.Map[String, Integer] = new util.HashMap[String, Integer]()
  topicCountMap.put(topic, new Integer(numOfConsumers))
  val consumerMap = consumerConnector.createMessageStreams(topicCountMap)
  consumerMap.get(topic).asScala.toArray
}
```

使用

```scala
    val streamIterator = kafkaConsumer map (_.iterator())

    while (true) {
      try {
        // 每个 consumer 收到的数据, 其实只有一个
        streamIterator foreach {
          // 每个 consumer 收到的数据
          case iterator: ConsumerIterator[Array[Byte], Array[Byte]] =>

            println("fetched size is: " + iterator.toList.size)
            while (iterator.hasNext()) {
              val message = iterator.next()
              val id: String = new String(message.key())
            }
        }
      } catch {
        case e: Exception =>
      }
    }
```

### Kafka Restful Wrapper

**超时返回:**

考虑这么一个场景, client 端从 kafka 拿数据, client 最多等 10s, 10s client 把收到的数据全部拿走,
10s 之后的数据它不再关心了。然而 kafka 的代码往往是 while 循环, 那么怎么处理这个问题呢

要点是, kafka 的 iterator 函数应该变成超时返回的, 先把 consumer.timeout.ms 设置成一个比较小的值, 然后
封装 next 方法, 搞一个 timedHasNext 

```scala
def timedHasNext(it: ConsumerIterator[Array[Byte], Array[Byte]]): Boolean = {
  try {
    it.hasNext()
    true
  } catch {
    case e: ConsumerTimeoutException =>
      log.error("consumer.timeout.ms might be to small", e)
      false
    case _ =>
      false
  }
}
```

```scala

val streamIterator = consumer map (_.iterator())

// 拿 size 个 message
try 
   (1 to size).foreach { _ =>
    streamIterator foreach { ci: ConsumerIterator[Array[Byte], Array[Byte]] => {
       if (System.currentTimeMillis() < requestEndTime && KafkaUtils.timedHasNext(ci)) {

        //if next throw exception, actor returns immediately
        val message = ci.next()

        val key = if(message.key() == null) null else new String(message.key)
        val value = new String(message.message())

        log.debug(s"consumer actor get message: ($key, $value) for topic: $topic")
        lstbf += MessageContent(key, value)
catch 
   case e: Exception =>
       log.error("failed to execute next() method", e)
finally 
   requester ! ConsumerResult(topic, group, lstbf.toList)
```

这样就能实现超时返回

**Providing Restful Service:**


## Zookeeper 信息

```
[zk: localhost:2181(CONNECTED) 18] get /consumers/qhhixjrDx/
offsets   owners    ids

// offsets
[zk: localhost:2181(CONNECTED) 18] get /consumers/qhhixjrDx/offsets/wrapper/0
24143 // 每次增一, 起始不定

// owner
[zk: localhost:2181(CONNECTED) 19] get /consumers/qhhixjrDx/owners/wrapper/0
qhhixjrDx_com-dev-service-01-1473672667169-57b00f16-0
```

Kafka partition has owner, so it cannot be shared within nodes. but why?

## 手动更新 Kafka 中某个 topic 的偏移量

### 更新 earliest, largest

```shell
$ bin/kafka-run-class.sh kafka.tools.UpdateOffsetsInZK 
USAGE: kafka.tools.UpdateOffsetsInZK$ [earliest | latest] consumer.properties topic
```

consumer.properties 中的配置是 zookeeper 和 group 信息

```
zookeeper.connect=www.iteblog.com:2181
 
# timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=6000

#consumer group id
```

但某些场景下, 这个工具不能满足我们的需求, 我们想手动设置成偏移量

### zookeeper 设置偏移量

Kafka topic 的偏移量一般都是存储在 Zookeeper 中, 具体的路径为 
`/consumers/[groupId]/offsets/[topic]/[partitionId]`

所以可以使用 set 来设置分区的偏移量

```
get /consumers/group/offsets/topic/0
70332526

set /consumers/group/offsets/topic/0 1024
```

## 如何选择 Topics Partitions 数量

越多的分区可以提供更高的吞吐量, 越多的分区需要打开更多的文件句柄, 更多的分区导致更高的不可用

越多的分区可能增加端到端的延迟, 越多的 partition 意味着客户端需要更大的内存

[link](http://www.confluent.io/blog/how-to-choose-the-number-of-topicspartitions-in-a-kafka-cluster/)

## Key 为 null 时, Kafka 如何选择分区 (partition)

我们往 kafka 发送消息时一般都会讲消息封装到 keyedMessage 类中:

```scala
val message = new KeyedMessage[String, String](topic, key, content)
producer.send(message)
```

kafka 会根据传进来的 key 计算 partition ID, 但是这个 key 可以不传或者为 null, 根据 kafka 的官方文档,
当 key 为 null 时, Producer 会将这个消息发送给随机的一个 Partition.

If the key is null, then the producer will assign the message to a random partition.

```scala
def getPartition(topic: String, key: Any, topicPartitionList: Seq[PartitionAndLeader]
    val numPartitions = topicPartitionList.size
    if(numParitions <= 0) throw new UnknownTopicPartitionExecption("Topic " + Topic + " doesn't exist")
    
    val partition =
        if(key == null)
        // if the key is null, we don't really need a partitioner
        // so we look up in the send partition cache for the topic to 
        // decide the target partition
        val id = sendPartitionPerTopicCache.get(topic)
        
        id match
            case Some(partitionId) =>
                // directly return the partitionId without checking availability
                // of the leader, since we want to postpone the failure until
                // the send operation anyway
                partitionId
            case None =>
                val availablePartitions = topicPartitionList.filter(_.leaderBrokerIdOpt.isDefined)
                if(availableParitions.isEmpty)
                    throw new LeaderNotAvailableException("No leader for any partition in topic " + topic)
                val index = Untis.abs(Random.nextInt) % availablePartitions.size
                val partitionId = avaiablePartitions(index).partitionId
                sendPartitionPerTopicCache.put(topic, partitionId)
                partitionId
        else
            partitioner.partition(key, numPartitions)
     if(partition < 0 || partition >= numPartitions)
        throw new UnknownTopicOrParititionExcepiton()
     
     partition
```

从代码可以看出来, 如果 Key == null, 则从 sendPartitionPerTopicCache (HashMap 类型) 获取分区 ID, 如果找到了就直接
获取分区 ID, 否则随机选择一个 partitionId, 并将 partitionId 存放到 sendPartitionPerTopicCache 中去。而且
`sendPartitionPerTopicCache` 是每隔 topic.metadata.refresh.interval.ms 时间才会清空的

也就是说在 Key == null 的情况下, kafka 并不是每条消息都随机选择一个 Partition, 没事每隔 interval.ms 才会随机一次, 这是
为了减少服务器端的 sockets 数(当 producer 远大于 brokers 时?)。但在 0.8.2 又被改回去了。总之, Key = null 时, 随机分配 partition.

新的算法是使用 Round-robin 每次选取一个分区 ID

```scala
if(record.partition() != null)
    if(record.partition() < 0 || record.partition() >= numPartitions)
        throw new IllegalArgumentException()
    else if (record.key() == null)
        int nextValue = counter.getAndIncrement()
        List<PartitionInfo> availablePartitions = cluster.avaiableParitionForTopic(record.topic)
        if(avaialblePartitions.size > 0)
            int part = Utils.abs(nextValue) % availablePartitions.size
            return avaiblesPartitions.get(part).partition
        else
            // no partition available, reutrn a non-avaiable partition
            return Utils.abs(nextValue) % numPartitions
    else
        return Utils.abs(Utils.murmur2(record.key) % numPartitions
```

### Message Delivery Semantics

Kafka's semantics are straight-forward. When publishing a message we have a notion of the 
message being "committed" to the log. Once a published message is committed it will not be lost 
as long as one broker that replicates the partition to which this message was written remains "alive".



So effectively Kafka guarantees at-least-once delivery by default and allows the user to implement at 
most once delivery by disabling retries on the producer and committing its offset prior to processing a 
batch of messages. Exactly-once delivery requires co-operation with the destination storage system 
but Kafka provides the offset which makes implementing this straight-forward.

### Replication

All reads and writes go to the leader of the partition.

Followers consume messages from the leader just as a normal Kafka consumer would and apply them to their 
own log. Having the followers pull from the leader has the nice property of allowing the follower to 
naturally batch together log entries they are applying to their log.

There are a rich variety of algorithms in this family including ZooKeeper's Zab, Raft, and Viewstamped Replication. 
The most similar academic publication we are aware of to Kafka's actual implementation is PacificA from Microsoft.

The downside of majority vote is that it doesn't take many failures to leave you with no electable leaders. 
To tolerate one failure requires three copies of the data, and to tolerate two failures requires five copies of 
the data. In our experience having only enough redundancy to tolerate a single failure is not enough for a 
practical system, but doing every write five times, with 5x the disk space requirements and 1/5th the throughput, 
is not very practical for large volume data problems. This is likely why quorum algorithms more commonly appear 
for shared cluster configuration such as ZooKeeper but are less common for primary data storage. For example 
in HDFS the namenode's high-availability feature is built on a majority-vote-based journal, but this 
more expensive approach is not used for the data itself.

**in-sync replicas (ISR)**

Kafka takes a slightly different approach to choosing its quorum set. Instead of majority vote, 
Kafka dynamically maintains a set of in-sync replicas (ISR) that are caught-up to the leader. Only members of this set 
are eligible for election as leader. A write to a Kafka partition is not considered committed until all in-sync 
replicas have received the write. This ISR set is persisted to ZooKeeper whenever it changes. Because of this, 
any replica in the ISR is eligible to be elected leader. This is an important factor for Kafka's usage model 
where there are many partitions and ensuring leadership balance is important. With this ISR model and f+1 replicas, 
a Kafka topic can tolerate f failures without losing committed messages.

### 问题1：发到哪个partition，谁来定?
    
在发送一条消息时，可以指定这条消息的key，Producer根据这个key和Partition机制来判断应该将这条消息发送到哪个Parition。
Paritition机制可以通过指定Producer的paritition. class这一参数来指定，该class必须实现kafka.producer.Partitioner接口。
本例中如果key可以被解析为整数则将对应的整数与Partition总数取余，该消息会被发送到该数对应的Partition。（每个Parition都会有个序号,序号从0开始）
    
这种问题没有正确的答案，只有到底在牺牲谁的答案。

在目前0.8.2.1的Kafka中，是交由Producer来解决这个问题的，Producer中有个 PartitionManager 专门用于负责对每个Message分配partition，或者由使用者更改。

**优势**

这样的优势在于Kafka Server不需要单独一个LoadBalancer来决定消息去哪里。而且Producer完全可以根据partition的id在ZK里寻找当前Leader，直接与Leader建立连接。

**劣势**

是不是看到这里发现问题了？是的，如果某个Partition完全不可用，这些消息就无法发送了。使用更加简化的模型带来的代价是牺牲了一部分可用性。

当然再有了副本策略之后，使一个partition变得不可用是一件很困难的事情。

### 问题2：日志如何存储？

Kafka Producer在确定partition leader之后开始与其所在的broker通信。为了使用磁盘的顺序写，即使用Log Structure storage。

为查找方便，Kafka同样建立了基本的索引结构。想想查询需求，有什么查询需求？大部分消息都会被顺序读取，当然也会存在少量的随机读取消息（比如处理的时候这条消息处理失败，需要重新处理）。所以索引在这里的意义仅为简单支持少量随机查询。

所以在索引的实现上，基本上就是为了支持针对某个Offset进行二分查找而存在的索引。

所以在文件存储上，每个消息被写成了两部分，一部分是『消息实体』，一部分是『消息索引』。消息实体格式如下：

```
message length ： 4 bytes (value: 1+4+n)
"magic" value ： 1 byte 
crc ： 4 bytes 
payload ： n bytes 
```

![](/images/posts/kafka/kafka_message_format.png)

### 问题3：如何实现日志副本&&副本策略&&同步方式
    
日志副本策略是可靠性的核心问题之一，其实现方式也是多种多样的。包括无主模型，通过paxos之类的协议保证消息顺序，
但更简单直接的方式是使用主从结构，主决定顺序，从拷贝主的信息。

如果主不挂，从节点没有存在的意义。但主挂了时，我们需要从备份节点中选出一个主。与此同时，更重要的是：保证一致性。在这里一致性是指:

1. 主ack了的消息，kafka切换主之后，依然可被消费。
2. 主没有ack的消息，kafka切换主之后，依然没有被存储。

因此这里产生了一个trade off：Leader应该什么时候ack呢？

这个问题简直是分布式环境里永恒（最坑爹）的主题之一了。其引申出的本质问题是，你到底要什么？

**要可靠性**

当然可以，leader收到消息之后，等follower 返回ok了ack，慢死。但好处是，主挂了，哪个follower都可以做主，大家数据都一样嘛

**要速度**

当然可以，leader收到消息写入本地就ack，然后再发给follower。问题也很显而易见，最坏得情况下，有个消息leader返回ack了，但follower因为各种原因没有写入，主挂了，丢数据了。

**副本问题的集中解决方式**

方式1: Quorum及类似协议

Quorum直译为决议团，即通过写多份，读多次的方式保证单个值是最新值，通俗理解为抽屉原理。

抽屉原理的适用范围很广，Dynamo在某种程度上也是使用了抽屉原理

在Dynamo的使用中，设共有N副本，每次写保证W个副本ack，每次读的时候读R个副本并从中取最新值，则只要保证W+R>N,那么就一定能保证读到最新数据。但那是在Key-Value的存储中使用的，没有数据顺序问题。在Kafka里，我们还需要有一个数据顺序问题。

Kafka中会持续写入数据，主接收数据后，向所有follower发送数据。当然，因为网络问题，每次成功ack的follower可能不完全相同，但可以当有W个节点ack的时候就进行主的ack。

这样，在主挂的时候，需要R个点共同选主，因为W+R>N，所以对于每条消息，R个点里一定是有一个点是写成功的。因此通过这R个点，一定可以拼凑出来一份齐全的，和Leader一样的数据。把这些数据写入单点，即可实现选新主。

当然，这里隐含了一个协议，就是leader每次向follower发送消息的时候是附带了消息编号的，且消息编号自增。里面还有很多实现细节(such as precisely defined what makes a log more complete, ensuring log consistency during leader failure or changing the set of servers in the replica set)，因为Kafka没用这种方式实现，所以也就不再复述。

**优势：**
这个方式的优势是，在写入过程中，跳过了部分反应慢的节点。因为要求W+R>N，所以选主速度应该也还可以。

**劣势：**性能较差，拥有2f+1的机器只能支持最多f台机器挂掉。假设我希望支持2台机器挂掉，我就需要5台机器。使用5台机器的存储，但只能存储一台机器的容量，以及一台机器的吞吐，这显然不是一个划算的买卖。

下面介绍ISR机制

1. 主挂掉的时候，直接从ISR里选一个当主。
2. 挂掉的主启动后，查看自己是不是主，不是主，当从。
3. ISR里记录了当前跟主一致的从节点，因此每次主收到消息后，需要等到ISR里所有机器ack了这个消息后才能对client进行ack。
4. 应该有探测机制动态的使从节点加入或者离开ISR，但文档没说。

### Kafka实现细节3:消费怎么保证不丢数据？

Kafka的高吞吐很大程度上得益于其放弃了对消费者offset的维护，而是放由消费者自行维护。

因此在消费者看来，kafka更像是一个专业顺序存储工具，而非一个消息队列。

问题1： Offset怎么存？

同事，Kafka提供了一种可以参考的方式的Offset存储方式。如果使用High-level Consumer，则可采用这种方式。

这种方式最有意思之处，是它用producer和consumer实现了一套可靠消息的Consumer，方式如下：

1. 对于每个UserGroup，Kafka会生成一个Offset Manager用于Handle所有partition的offset。Manager实际上通过一个内建的compacted topic（叫做consumer）
2. 所有Consumer都要发送其offset到offset topic。
3. 当Consumer要消费时，先去offset topic取出最新的对应消息，然后消费。

### 问题3： Consumer的关闭异常，会不会存在Offset异常导致多消费或者少消费？
    
事实上是的。每当Consumer被异常重启时，有一定几率会有一部分数据被重复消费，或者被跳过。

重复数据的数量取决于Consumer同步的频率。比如：

Consumer每1k条消息进行一次消息同步，consumer消费到43k时，又消费了300条，然后跪了。

Consumer Manager只记录到了43k

Consumer重启了，读自己UserGroup里Consumer Manger在该partition最新一个消息：43k。再读一下自己UserGroup里数据量：43.3k

这时，他有两个选择

1. 按照43k的offset继续读，那么之前的300条消息被重复消费了
2. 按照43.3读，那么有可能那300条消息之前没消费，所以对了300条

当然会问，会不会有Exactly Once语义？

答案是，你自己来实现吧。解法:

只要保证consumer里留一些缓存，缓冲一批消息之后，做一个transaction:

1. 向后端发送数据的同时
2. 向producer发送汇报

即可保证Exactly Once语义










