---
layout: post
title:  "Kafka walk through"
date:   "2016-09-12 00:00:00"
categories: Kafka
keywords: Kafka
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



