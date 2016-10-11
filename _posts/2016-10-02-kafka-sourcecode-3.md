---
layout: post
title: kafka source code3
categories: [kafka]
keywords: kafka
---

## Fast Path

结合上一篇 post 的内容

Consumer 会先创建 ConsumerConnector, 它代表一个消费者进程, 包含两个方法, 分别是 commitOffset 和 createStream

createStream 是让 developer 调用创建流的, commitOffset 一般不需要 dev 管理, connector

ConsumerConnector 的实现类型是 ZookeeperConsumerConnector, 因为 client 需要频繁连接 zookeeper 并发送消息。consumerConnector 
创建时, 会启动 ZookeeperClient, ConsumerFetcherManager, 其中 ConsumerFetcherManager 就是用来取数据的, 当然 ConsumerConnector
还要处理消费者到来/离开, partition增减等事情。这个先不说

ConsumerFetcherManager 管理一组 FetcherThread, 对于用户感兴趣的 topic, 都有会 topicPartition 到 fetcherThread 的映射, 
在 Producer 中, 每个 broker 一个 channel, 但是 consumer 每个 partition 一个 channel. 除此之外, ConsumerFetcherManager
还维护了一个 map, 它是 topicPartition 到 **Partition 数据**的映射, 数据的类型是 PartitionTopicInfo, 除了数据意外, 它还保存着
consumedOffset 以及 fetchedOffset 和 clientId, kafkaStream 中的数据就是从 PartitionTopicInfo 中拉取的

ConsumerFetcherManager 除了获取数据外, 还维护了一个 LeaderFindThread, 这个线程的作用就是不停的和 zookeeper 通信, 获取
当前感兴趣的 topic partition primary 节点所在的位置, 当发生变化时, 更新自己的 topic, partition 到 fetcherThread 的映射。
值得注意的是, Replica partition 只负责 HA 不负责 LB, 所以 Replica 也有自己的 FetcherManager 和 FetcherThread

FetcherManager 中的 FindLeadThread 用于处理 partition 的增加减少, 对于 Consumer 的加入离开, 由 ConsumerConnector 来处理,
因为消费者加入/退出时,消费组的成员会发生变化,而消费组中的所有存活消费者负责消费可用的partitions.

可用的partitions或者消费组中的消费者成员一旦发生变化,都要重新分配partition给存活的消费者

- PartitionOwnership: Partition的所有者(ownership)的删除和重建
- AssignmentContext: 分配信息上下文
- PartitionAssignor: 为 Partition 分配 Consumer 的算法
- PartitionAssignment: Partition 分配之后的上下文
- PartitionTopicInfo: Partition 的最终信息
- Fetcher: 完成了 rebalance, 消费者就可以重新开始抓取数据

关于 consumerConnector 是 consumer 组还是 consumer 个体, 可以参考它这个成员变量

```scala
private val topicThreadIdAndQueues = new Pool[(String, ConsumerThreadId), BlockingQueue[FetchedDataChunk]]
```

Key 是 Topic, ConsumerThreadId 是 consumerId 和 threadId, 所以说, consumerConnector 应该是 consumer 组的概念,
当有多个 consumer 进程时, 一个 consumer 进程离开了, 其他的 consumerConnector 应该都会发生变化, 那么一个 consumer 是怎么
知晓其他 consumer 的存在的呢? 这是因为每个 consumer 都会监听 zookeeper 中的 consumer/consumerGroup/owners 信息吧

```scala
class ZKRebalancerListener(val group: String, val consumerIdString: String,
                             val kafkaMessageAndMetadataStreams: mutable.Map[String,List[KafkaStream[_,_]]])
    extends IZkChildListener
```

## Consumer(2) Fetcher

获取TopicMetadata,使用生产者模式发送一个需要响应结果的TopicMetadataRequest.
因为一个topic分成多个partition,所以一个TopicMetadata包括多个PartitionMetadata.
PartitionMetadata表示Partition的元数据,有Partition的Leader信息.

```scala
def fetchTopicMetadata(topics: Set[String], brokers: Seq[BrokerEndPoint], 
    clientId: String, timeoutMs: Int, correlationId: Int = 0): TopicMetadataResponse = {
  
  val props = new Properties()
  props.put("metadata.broker.list", brokers.map(_.connectionString).mkString(","))
  props.put("client.id", clientId)
  props.put("request.timeout.ms", timeoutMs.toString)
  val producerConfig = new ProducerConfig(props)
  fetchTopicMetadata(topics, brokers, producerConfig, correlationId)
}
def fetchTopicMetadata(topics: Set[String], brokers: Seq[BrokerEndPoint], 
    producerConfig: ProducerConfig, correlationId: Int): TopicMetadataResponse = {
  
  var fetchMetaDataSucceeded: Boolean = false
  var i: Int = 0
  val topicMetadataRequest = new TopicMetadataRequest(TopicMetadataRequest.CurrentVersion, 
    correlationId, producerConfig.clientId, topics.toSeq)
  var topicMetadataResponse: TopicMetadataResponse = null
  // shuffle the list of brokers before sending metadata requests so that most requests don't get routed to the same broker
  val shuffledBrokers = Random.shuffle(brokers)
  // 随机向一个Broker 发送 Producer 请求,只要成功一次后,就算成功了
  while(i < shuffledBrokers.size && !fetchMetaDataSucceeded) {
    val producer: SyncProducer = ProducerPool.createSyncProducer(producerConfig, shuffledBrokers(i))
    try {
      topicMetadataResponse = producer.send(topicMetadataRequest)
      fetchMetaDataSucceeded = true
    } finally {
      i = i + 1
      producer.close()
    }
  }
  topicMetadataResponse
}
```

LeaderFinderThread负责在Leader Partition可用的时候,将Fetcher添加到正确的Broker.
这里addFetcherForPartitions明明是为Partition添加Fetcher,为什么说是添加到Broker?
因为addFetcherForPartitions会创建FetcherThread是以(fetcherId,brokerId)为粒度的.

```scala
private class LeaderFinderThread(name: String) extends ShutdownableThread(name) {
  override def doWork() {
      val leaderForPartitionsMap = new HashMap[TopicAndPartition, BrokerEndPoint]
      while (noLeaderPartitionSet.isEmpty) cond.await()  // No partition for leader election

      val brokers = zkUtils.getAllBrokerEndPointsForChannel(SecurityProtocol.PLAINTEXT)
      val topicsMetadata = ClientUtils.fetchTopicMetadata(noLeaderPartitionSet.map(m => m.topic).toSet, 
            brokers,config.clientId,config.socketTimeoutMs, correlationId.getAndIncrement).topicsMetadata
      topicsMetadata.foreach { tmd =>             // TopicMetadata
        tmd.partitionsMetadata.foreach { pmd =>   // PartitionMetadata
          val topicAndPartition = TopicAndPartition(tmd.topic, pmd.partitionId)
          // Partition存在Leader,而且存在于noLeaderPartitionSet中,则从noLeaderPartitionSet中移除,并且加到新的map中
          if(pmd.leader.isDefined && noLeaderPartitionSet.contains(topicAndPartition)) {
            leaderForPartitionsMap.put(topicAndPartition, pmd.leader.get)
            noLeaderPartitionSet -= topicAndPartition
          }
        }
      }
      // 要为Fetcher添加的Partitions是有Leader的: TopicAndPartition->BrokerInitialOffset
      addFetcherForPartitions(leaderForPartitionsMap.map{
        // (rebalance)partitionMap是topicRegistry的PartitionTopicInfo, 包含了fetchOffset和consumerOffset
        case (topicAndPartition, broker) => topicAndPartition -> 
          BrokerAndInitialOffset(broker, partitionMap(topicAndPartition).getFetchOffset())}
      )
      shutdownIdleFetcherThreads()    // 上面为Fetcher添加partition,如果fetcher没有partition,则删除该fetcher.
      Thread.sleep(config.refreshLeaderBackoffMs)
  }
}
```

leaderForPartitionsMap的映射关系是TopicAndPartition到leaderBroker. 抓取数据要关心Partition的offset,从Partition的哪个offset开始抓取数据.

### AbstractFetcherManager

每个消费者都有自己的ConsumerFetcherManager.fetch动作不仅只有消费者有,Partition的副本也会拉取Leader的数据.

createFetcherThread抽象方法对于Consumer和Replica会分别创建ConsumerFetcherThread和ReplicaFetcherThread.

由于消费者可以消费多个topic的多个partition.每个TopicPartition组合都会有一个fetcherId.

所以fetcherThreadMap的key实际上是由(broker_id, topic_id, partition_id)共同组成的.

针对每个source broker的每个partition都会有拉取线程,即拉取是针对partition级别拉取数据的.

```scala
abstract class AbstractFetcherManager(protected val name: String, clientId: String, numFetchers: Int = 1) {
  // map of (source broker_id, fetcher_id per source broker) => fetcher
  private val fetcherThreadMap = new mutable.HashMap[BrokerAndFetcherId, AbstractFetcherThread]
}
case class BrokerAndFetcherId(broker: BrokerEndPoint, fetcherId: Int)
case class BrokerAndInitialOffset(broker: BrokerEndPoint, initOffset: Long)
```

所以BrokerAndFetcherId可以表示Borker上某个topic的PartitionId,而BrokerAndInitialOffset是Broker级别的offset.

addFetcherForPartitions的参数中BrokerAndInitialOffset是和TopicAndPartition有关的,即Partition的offset.

```scala
// to be defined in subclass to create a specific fetcher
def createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint): AbstractFetcherThread

// 为Partition添加Fetcher是为Partition创建Fetcher线程. 因为Fetcher线程是用来抓取Partition的消息. 
def addFetcherForPartitions(partitionAndOffsets: Map[TopicAndPartition, BrokerAndInitialOffset]) {
    // 根据broker-topic-partition分组. 相同fetcherId对应一个fetcher线程  
    val partitionsPerFetcher = partitionAndOffsets.groupBy{ case(topicAndPartition, brokerAndInitialOffset) =>
      BrokerAndFetcherId(brokerAndInitialOffset.broker, getFetcherId(topicAndPartition.topic, topicAndPartition.partition))}
    // 分组之后的value仍然不变,还是partitionAndOffsets. 不过因为是根据fetcherId,可能存在不同的partition有相同的fetcherId 
    for ((brokerAndFetcherId, partitionAndOffsets) <- partitionsPerFetcher) {
      var fetcherThread: AbstractFetcherThread = null
      fetcherThreadMap.get(brokerAndFetcherId) match {
        case Some(f) => fetcherThread = f
        case None =>
          fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
          fetcherThreadMap.put(brokerAndFetcherId, fetcherThread)
          fetcherThread.start         // 启动刚刚创建的拉取线程
      }

      // 由于partitionAndOffsets现在已经是在同一个partition里. 取得所有partition对应的offset
      fetcherThreadMap(brokerAndFetcherId).addPartitions(partitionAndOffsets.map { 
        case (topicAndPartition, brokerAndInitOffset) => topicAndPartition -> brokerAndInitOffset.initOffset
      })
    }
}
```

partitionAndOffsets.groupBy的partitionAndOffsets是包括所有Broker-Topic-Partition的.

分组后,for循环中partitionsPerFetcher的partitionAndOffsets对相同fetcherId的partitions会被分到同一组.

在同一个for循环里的partitionAndOffsets有相同的fetcherId,可能会有多个partitionAndOffsets.

**FetcherManager** 管理所有的FetcherThread,而每个FetcherThread则管理自己的PartitionOffset.

每个角色都各司其职,管理者不需要关心底层的Partition,而是交给线程来管理,因为线程负责处理Partition.
这就好比常见的Master-Worker架构,Master是管理者,负责管理所有的Worker进程,而Worker负责具体的Task.

### AbstractFetcherThread addPartitions

Consumer和Replica的FetcherManager都会负责将自己要抓取的partitionAndOffsets传给对应的Fetcher线程.

Consumer交给LeaderFinderThread发现线程, Replica则是在makeFollowers时确定partitionOffsets.

```scala
AbstractFetcherManager.addFetcherForPartitions(Map<TopicAndPartition, BrokerAndInitialOffset>)  (kafka.server)
    |-- LeaderFinderThread in ConsumerFetcherManager.doWork()  (kafka.consumer)
    |-- ReplicaManager.makeFollowers(int, int, Map<Partition, PartitionState>, int, Map<TopicPartition, Object>, MetadataCache)  (kafka.server)
```

抓取线程也是用partitionMap缓存来保存每个TopicAndPartition的抓取状态.即管理者负责线程相关,而线程负责状态相关.

```scala
// Abstract class for fetching data from multiple partitions from the same broker.
abstract class AbstractFetcherThread(name: String, clientId: String, 
    sourceBroker: BrokerEndPoint, fetchBackOffMs: Int = 0, isInterruptible: Boolean = true) {
  
  private val partitionMap = new mutable.HashMap[TopicAndPartition, PartitionFetchState] // a (topic, partition) -> partitionFetchState map

  def addPartitions(partitionAndOffsets: Map[TopicAndPartition, Long]) {
    partitionMapLock.lockInterruptibly()
    try {
      for ((topicAndPartition, offset) <- partitionAndOffsets) {
        // If the partitionMap already has the topic/partition, then do not update the map with the old offset
        if (!partitionMap.contains(topicAndPartition))
          partitionMap.put(topicAndPartition,
            if (PartitionTopicInfo.isOffsetInvalid(offset)) new PartitionFetchState(handleOffsetOutOfRange(topicAndPartition))
            else new PartitionFetchState(offset)
          )}
      partitionMapCond.signalAll()
    } finally partitionMapLock.unlock()
  }
```

总结下从消费者通过ZKRebalancerListener获取到分配给它的PartitionAssignment,转换为topicRegistry.
交给AbstractFetcherManager管理所有的FetcherThread. 同时有Leader发现线程获取Partition的Leader.
Partition的offset的源头是topicRegistry的fetchOffsets(即从offsetChannel获取),贯穿于整个流程.

![](http://img.blog.csdn.net/20160129084955562)

### KafkaApis 的处理逻辑



