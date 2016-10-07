---
layout: post
title: Kafka Consumer
categories: [kafka]
keywords: kafka, java, nio
---

### All start from ConsumerConnector

ConsumerConnector 做两件事, 一件事是 createStream 另一件事提交 offset, 它是一个 consumer 的大总管

```scala
public interface ConsumerConnector {
    public <K,V> Map<String, List<KafkaStream<K,V>>> 
        createMessageStreams(Map<String, Integer> topicCountMap, Decoder<K> keyDecoder, Decoder<V> valueDecoder);
    
    public void commitOffsets();
```

它的唯一实现子类是 ZookeeperConsumerConnector

```scala
 * This class handles the consumers interaction with zookeeper
 *
 * Directories:
 * 1. Consumer id registry:
 * /consumers/[group_id]/ids[consumer_id] -> topic1,...topicN
 * A consumer has a unique consumer id within a consumer group. A consumer registers its id as an ephemeral znode
 * and puts all topics that it subscribes to as the value of the znode. The znode is deleted when the client is gone.
 * A consumer subscribes to event changes of the consumer id registry within its group.
 *
 * The consumer id is picked up from configuration, instead of the sequential id assigned by ZK. Generated sequential
 * ids are hard to recover during temporary connection loss to ZK, since it's difficult for the client to figure out
 * whether the creation of a sequential znode has succeeded or not. More details can be found at
 * (http://wiki.apache.org/hadoop/ZooKeeper/ErrorHandling)
 *
 * 2. Broker node registry:
 * /brokers/[0...N] --> { "host" : "host:port",
 *                        "topics" : {"topic1": ["partition1" ... "partitionN"], ...,
 *                                    "topicN": ["partition1" ... "partitionN"] } }
 * This is a list of all present broker brokers. A unique logical node id is configured on each broker node. A broker
 * node registers itself on start-up and creates a znode with the logical node id under /brokers. The value of the znode
 * is a JSON String that contains (1) the host name and the port the broker is listening to, (2) a list of topics that
 * the broker serves, (3) a list of logical partitions assigned to each topic on the broker.
 * A consumer subscribes to event changes of the broker node registry.
 *
 * 3. Partition owner registry:
 * /consumers/[group_id]/owner/[topic]/[broker_id-partition_id] --> consumer_node_id
 * This stores the mapping before broker partitions and consumers. Each partition is owned by a unique consumer
 * within a consumer group. The mapping is reestablished after each rebalancing.
 *
 * 4. Consumer offset tracking:
 * /consumers/[group_id]/offsets/[topic]/[broker_id-partition_id] --> offset_counter_value
 * Each consumer tracks the offset of the latest message consumed for each partition.
 *
*/

private[kafka] class ZookeeperConsumerConnector(val config: ConsumerConfig) extends ConsumerConnector {
    private var fetcher: Option[ConsumerFetcherManager] = None
    private var zkClient: ZkClient = null
    private var topicRegistry = new Pool[String, Pool[Int, PartitionTopicInfo]]
    private val checkpointedZkOffsets = new Pool[TopicAndPartition, Long]
    private val topicThreadIdAndQueues = new Pool[(String, ConsumerThreadId), BlockingQueue[FetchedDataChunk]]
    
    private val scheduler = new KafkaScheduler(threads = 1, threadNamePrefix = "kafka-consumer-scheduler-")
    
    private val messageStreamCreated = new AtomicBoolean(false)
    
    private var sessionExpirationListener: ZKSessionExpireListener = null
    private var topicPartitionChangeListener: ZKTopicPartitionChangeListener = null
    private var loadBalancerListener: ZKRebalancerListener = null
    
    private var offsetsChannel: BlockingChannel = null
    private val offsetsChannelLock = new Object

    private var wildcardTopicWatcher: ZookeeperTopicEventWatcher = null
    
    val consumerIdString = {
        consumerUuid = "%s-%d-%s".format(
                  InetAddress.getLocalHost.getHostName, System.currentTimeMillis,
                  uuid.getMostSignificantBits().toHexString.substring(0, 8))
        config.groupId + "_" + consumerUuid }
        
    connectZk()
    createFetcher()
    ensureOffsetManagerConnected()

    if (config.autoCommitEnable) {
        scheduler.startup
        info("starting auto committer every " + config.autoCommitIntervalMs + " ms")
        scheduler.schedule("kafka-consumer-autocommit",
          autoCommit,
          delay = config.autoCommitIntervalMs,
          period = config.autoCommitIntervalMs,
          unit = TimeUnit.MILLISECONDS) }
    
    AppInfo.registerInfo()
```

一个Consumer会创建一个ZookeeperConsumerConnector,代表一个消费者进程.

1. fetcher: 消费者获取数据, 使用ConsumerFetcherManager fetcher线程抓取数据
2. zkUtils: 消费者要和ZK通信, 除了注册自己,还有其他信息也会写到ZK中
3. topicThreadIdAndQueues: 消费者会指定自己消费哪些topic,并指定线程数, 所以topicThreadId都对应一个队列
4. messageStreamCreated: 消费者会创建消息流, 每个队列都对应一个消息流
5. offsetsChannel: offset可以存储在ZK或者kafka中,如果存在kafka里,像其他请求一样,需要和Broker通信
6. 还有其他几个Listener监听器,分别用于topicPartition的更新,负载均衡,消费者重新负载等

```scala
def consume[K, V](topicCountMap: scala.collection.Map[String,Int], 
  keyDecoder: Decoder[K], valueDecoder: Decoder[V]) : Map[String,List[KafkaStream[K,V]]] = {
  
  val topicCount = TopicCount.constructTopicCount(consumerIdString, topicCountMap)
  val topicThreadIds = topicCount.getConsumerThreadIdsPerTopic

  // make a list of (queue,stream) pairs, one pair for each threadId 只是准备了队列和流,数据什么时候填充呢?
  val queuesAndStreams = topicThreadIds.values.map(threadIdSet =>
    threadIdSet.map(_ => {
      val queue =  new LinkedBlockingQueue[FetchedDataChunk](config.queuedMaxMessages)
      val stream = new KafkaStream[K,V](queue, config.consumerTimeoutMs, keyDecoder, valueDecoder, config.clientId)
      (queue, stream)
    })
  ).flatten.toList  //threadIdSet是个集合,外层的topicThreadIds.values也是集合,所以用flatten压扁为queue-stream对

  val dirs = new ZKGroupDirs(config.groupId)                  // /consumers/[group_id]
  registerConsumerInZK(dirs, consumerIdString, topicCount)    // /consumers/[group_id]/ids/[consumer_id]
  reinitializeConsumer(topicCount, queuesAndStreams)          // 重新初始化消费者 ⬅️

  // 返回KafkaStream, 每个Topic都对应了多个KafkaStream. 数量和topicCount中的count一样
  loadBalancerListener.kafkaMessageAndMetadataStreams.asInstanceOf[Map[String, List[KafkaStream[K,V]]]]
}
```

对于消费者而言,它只要指定要消费的topic和线程数量就可以了,其他具体这个topic分成多少个partition,
以及topic-partition是分布是哪个broker上,对于客户端而言都是透明的.
客户端关注的是我的每个线程都对应了一个队列,每个队列都是一个消息流就可以了.
在Producer以及前面分析的Fetcher,都是以Broker-Topic-Partition为级别的.
AbstractFetcherManager的fetcherThreadMap就是以brokerAndFetcherId来创建拉取线程的.

而消费者是通过拉取线程才有数据可以消费的,所以客户端的每个线程实际上也是针对Partition级别的.

### registerConsumerInZK

消费者需要向ZK注册一个临时节点,路径为: `/consumers/[group_id]/ids/[consumer_id]`,内容为订阅的topic.

```scala
private def registerConsumerInZK(dirs: ZKGroupDirs, consumerIdString: String, topicCount: TopicCount) {
  val consumerRegistrationInfo = Json.encode(Map("version" -> 1, 
    "subscription" -> topicCount.getTopicCountMap, "pattern" -> topicCount.pattern, "timestamp" -> timestamp))

  val zkWatchedEphemeral = new ZKCheckedEphemeral(dirs.consumerRegistryDir + "/" + consumerIdString, 
    consumerRegistrationInfo, zkUtils.zkConnection.getZookeeper, false)

  zkWatchedEphemeral.create()
}
```

问题:什么时候这个节点会被删除掉呢? Consumer进程挂掉时,或者Session失效时删除临时节点. 重连时会重新创建.

由于是临时节点,一旦创建节点的这个进程挂掉了,临时节点就会自动被删除掉. 这是由zk机制决定的,不是由消费者完成的.

当前Consumer在ZK注册之后,需要重新初始化Consumer. 对于全新的消费者,注册多个监听器,在zk的对应节点的注册事件发生时,会回调监听器的方法.

将topic对应的消费者线程id及对应的LinkedBlockingQueue放入topicThreadIdAndQueues中,LinkedBlockingQueue是真正存放数据的queue

1. 注册sessionExpirationListener,监听状态变化事件.在session失效重新创建session时调用
2. 向/consumers/[group_id]/ids注册Child变更事件的loadBalancerListener,当消费组下的消费者发生变化时调用
3. 向/brokers/topics/[topic]注册Data变更事件的topicPartitionChangeListener,在topic数据发生变化时调用
4. 显式调用loadBalancerListener.syncedRebalance(), 会调用reblance方法进行consumer的初始化工作

```scala
private def reinitializeConsumer[K,V](topicCount: TopicCount, 
  queuesAndStreams: List[(LinkedBlockingQueue[FetchedDataChunk],KafkaStream[K,V])]) {
  
  val dirs = new ZKGroupDirs(config.groupId)
  
  // ② listener to consumer and partition changes
  if (loadBalancerListener == null) {
    val topicStreamsMap = new mutable.HashMap[String,List[KafkaStream[K,V]]]
    loadBalancerListener = new ZKRebalancerListener(config.groupId, consumerIdString, 
      topicStreamsMap.asInstanceOf[scala.collection.mutable.Map[String, List[KafkaStream[_,_]]]])
  }
  // ① create listener for session expired event if not exist yet
  if (sessionExpirationListener == null) sessionExpirationListener = 
    new ZKSessionExpireListener(dirs, consumerIdString, topicCount, loadBalancerListener)
  
  // ③ create listener for topic partition change event if not exist yet
  if (topicPartitionChangeListener == null) 
    topicPartitionChangeListener = new ZKTopicPartitionChangeListener(loadBalancerListener)

  // listener to consumer and partition changes
  zkUtils.zkClient.subscribeStateChanges(sessionExpirationListener)
  zkUtils.zkClient.subscribeChildChanges(dirs.consumerRegistryDir, loadBalancerListener)
  
  // register on broker partition path changes.
  topicStreamsMap.foreach { topicAndStreams => 
    zkUtils.zkClient.subscribeDataChanges(BrokerTopicsPath+"/"+topicAndStreams._1, topicPartitionChangeListener)
  }

  // explicitly trigger load balancing for this consumer
  loadBalancerListener.syncedRebalance()
}
```

上面的代码没看出 KafkaStream 是如何被填充数据的

ZKRebalancerListener 传入 ZKSessionExpireListener 和 ZKTopicPartitionChangeListener. 
它们都会使用 ZKRebalancerListener 完成自己的工作

**ZKSessionExpireListener**, 当Session失效时,新的会话建立时,立即进行rebalance操作

**ZKTopicPartitionChangeListener**, 当topic的数据变化时,通过触发的方式启动rebalance操作.

### ZKRebalancerListener rebalance

因为消费者加入/退出时,消费组的成员会发生变化,而消费组中的所有存活消费者负责消费可用的partitions.

```scala
private def rebalance(cluster: Cluster): Boolean = {
  val myTopicThreadIdsMap = TopicCount.constructTopicCount(group, consumerIdString, 
    zkUtils, config.excludeInternalTopics).getConsumerThreadIdsPerTopic
    
  val brokers = zkUtils.getAllBrokersInCluster()
  
  if (brokers.size == 0) {
    zkUtils.zkClient.subscribeChildChanges(BrokerIdsPath, loadBalancerListener)
    true
  } else {
  
    // ① 停止fetcher线程防止数据重复.如果当前调整失败了,被释放的partitions可能被其他消费者拥有.
    // 而没有先停止fetcher的话,原先的消费者仍然会和新的拥有者共同消费同一份数据.  
    closeFetchers(cluster, kafkaMessageAndMetadataStreams, myTopicThreadIdsMap)
  
    // ② 释放topicRegistry中topic-partition的owner
    releasePartitionOwnership(topicRegistry)
    
    // ③ 为partition重新分配消费者....
    
    // ④ 为partition添加consumer owner
    if(reflectPartitionOwnershipDecision(partitionAssignment)) {
        allTopicsOwnedPartitionsCount = partitionAssignment.size
        topicRegistry = currentTopicRegistry
    
        // ⑤ 创建拉取线程
        updateFetcher(cluster)
        true    }}}
```

rebalance操作涉及了以下内容:

1. PartitionOwnership: Partition的所有者(ownership)的删除和重建
2. AssignmentContext: 分配信息上下文
3. PartitionAssignor: 为Partition分配Consumer的算法
4. PartitionAssignment: Partition分配之后的上下文
5. PartitionTopicInfo: Partition的最终信息
6. Fetcher: 完成了rebalance, 消费者就可以重新开始抓取数据

### PartitionOwnership

topicRegistry的数据结构是: `topic -> (partition -> PartitionTopicInfo)`, 表示现有的topic注册信息.

当partition被consumer所拥有后, 会在zk中创建`/consumers/[group_id]/owner/[topic]/[partition_id]` --> consumer_node_id

释放所有partition的ownership, 数据来源于topicRegistry的topic-partition(消费者所属的group_id也是确定的).

所以deletePartitionOwnershipFromZK会删除`/consumers/[group_id]/owner/[topic]/[partition_id]`节点.

这样partition没有了owner,说明这个partition不会被consumer消费了,也就相当于consumer释放了partition.

## ConsumerFetcherManager

![](/images/posts/kafka/FetcherProcess.png)

我觉得这张图不够好, 没有把工厂模式的设计模式体现出来

ConsumerFetcherManager管理了当前Consumer的所有Fetcher线程.

注意ConsumerFetcherThread构造函数中的partitionMap和构建FetchRequest时的partitionMap是不同的.

不过它们的相同点是都有offset信息.并且都有fetch操作.

```scala
abstract class AbstractFetcherManager(protected val name: String, clientId: String, numFetchers: Int = 1)
    private val fetcherThreadMap = new mutable.HashMap[BrokerAndFetcherId, AbstractFetcherThread]

    private def getFetcherId(topic: String, partitionId: Int): Int = {
        Utils.abs(31 * topic.hashCode() + partitionId) % numFetchers
    }
    
    // 为每个 partition 添加 FetcherThread    
    def addFetcherForPartitions(partitionAndOffsets: Map[TopicAndPartition, BrokerAndInitialOffset]) {
        mapLock synchronized {
          val partitionsPerFetcher = partitionAndOffsets.groupBy { case (topicAndPartition, brokerAndInitialOffset) =>
            BrokerAndFetcherId(brokerAndInitialOffset.broker, getFetcherId(topicAndPartition.topic, topicAndPartition.partition))
          }
    
          for ((brokerAndFetcherId, partitionAndOffsets) <- partitionsPerFetcher) {
            var fetcherThread: AbstractFetcherThread = null
            fetcherThreadMap.get(brokerAndFetcherId) match {
              case Some(f) => fetcherThread = f
              case None =>
                fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
                fetcherThreadMap.put(brokerAndFetcherId, fetcherThread)
                fetcherThread.start
            }
    
            fetcherThreadMap(brokerAndFetcherId).addPartitions(partitionAndOffsets.map { case (topicAndPartition, brokerAndInitOffset) =>
              topicAndPartition -> brokerAndInitOffset.initOffset
            })
          }
        }
    
        info("Added fetcher for partitions %s".format(partitionAndOffsets.map { case (topicAndPartition, brokerAndInitialOffset) =>
          "[" + topicAndPartition + ", initOffset " + brokerAndInitialOffset.initOffset + " to broker " + brokerAndInitialOffset.broker + "] "
        }))
      }

class ConsumerFetcherManager(private val consumerIdString: String,
                             private val config: ConsumerConfig,
                             private val zkClient : ZkClient)
        extends AbstractFetcherManager("ConsumerFetcherManager-%d".format(SystemTime.milliseconds),
                                       config.clientId, config.numConsumerFetchers) {
  
  private var partitionMap: immutable.Map[TopicAndPartition, PartitionTopicInfo] = null
  private var cluster: Cluster = null
  private var leaderFinderThread: ShutdownableThread = null // 更新 parition 的线程
  private val correlationId = new AtomicInteger(0)
```

因为 FetcherManager 有 ReplicaFetcherManager 和 ConsumerFetcherManager, 所以 createFetcherThread 的方法延迟到
子类实现, 这算是工厂方法吧

在 ConsumerFetcherManager 还有一个线程叫做 LeaderFinderThread, 它的作用就是定时的查找 partition 对应的 broker, 然后
更新当前节点知道的信息, 通过修改 fetcherThreadMap 和调用 addPartition 方法实现。 而 replicaManager 就没这个线程 (为什么?)

### LeaderFinderThread

```scala
override def doWork() {
    val leaderForPartitionsMap = new HashMap[TopicAndPartition, Broker]
    val topicsMetadata = ClientUtils.fetchTopicMetadata(noLeaderPartitionSet.map(m => m.topic).toSet,
        brokers,
        config.clientId,
        config.socketTimeoutMs,
        correlationId.getAndIncrement).topicsMetadata
    
    // 更新 fetch thread
    addFetcherForPartitions(leaderForPartitionsMap.map{
      case (topicAndPartition, broker) =>
        topicAndPartition -> BrokerAndInitialOffset(broker, partitionMap(topicAndPartition).getFetchOffset())})
```

### FetcherThread

FetcherThread 有 ConsumerFetcherThread 和 ReplicaFetcherThread 两种

```scala
abstract class AbstractFetcherThread(name: String, clientId: String, sourceBroker: Broker, socketTimeout: Int, socketBufferSize: Int,
                                     fetchSize: Int, fetcherBrokerId: Int = -1, maxWait: Int = 0, minBytes: Int = 1,
                                     isInterruptible: Boolean = true)
  extends ShutdownableThread(name, isInterruptible)
  
  private val partitionMap = new mutable.HashMap[TopicAndPartition, Long] // a (topic, partition) -> offset map
  val simpleConsumer = new SimpleConsumer(sourceBroker.host, sourceBroker.port, 
    socketTimeout, socketBufferSize, clientId)
  
  override def doWork() {
      inLock(partitionMapLock) {
        if (partitionMap.isEmpty)
          partitionMapCond.await(200L, TimeUnit.MILLISECONDS)
        partitionMap.foreach {
          case ((topicAndPartition, offset)) =>
            fetchRequestBuilder.addFetch(topicAndPartition.topic, topicAndPartition.partition,
              offset, fetchSize)
        }
      }
  
      val fetchRequest = fetchRequestBuilder.build()
      if (!fetchRequest.requestInfo.isEmpty)
        processFetchRequest(fetchRequest)
    }
  
  private def processFetchRequest(fetchRequest: FetchRequest) {
    var response: FetchResponse = null
    response = simpleConsumer.fetch(fetchRequest)
    if (response != null) {
        response.data.foreach {
            case (topicAndPartition, partitionData) =>
                val (topic, partitionId) = topicAndPartition.asTuple
                val currentOffset = partitionMap.get(topicAndPartition)
                if (currentOffset.isDefined && fetchRequest.requestInfo(topicAndPartition).offset == currentOffset.get) {
                    val messages = partitionData.messages.asInstanceOf[ByteBufferMessageSet]
                    val validBytes = messages.validBytes
                    val newOffset = messages.shallowIterator.toSeq.lastOption match {
                        case Some(m: MessageAndOffset) => m.nextOffset
                        case None => currentOffset.get }
                    partitionMap.put(topicAndPartition, newOffset)
                    processPartitionData(topicAndPartition, currentOffset.get, partitionData)
    if (partitionsWithError.size > 0) {
        debug("handling partitions with error for %s".format(partitionsWithError))
        handlePartitionsWithErrors(partitionsWithError)
    }                
```

对于 Consumer 和 Replica 有不同的 handlePartitionWithErrors 的实现, 其实在这两个子类中, 实现的方法都是各种 handle 方法,
包括对正确数据的进一步处理和错误信息的处理

ConsumerFetcherThread

```scala
pti: PartitionTopicInfo

def processPartitionData(topicAndPartition: TopicAndPartition, fetchOffset: Long, partitionData: FetchResponsePartitionData) {
    val pti = partitionMap(topicAndPartition)
    if (pti.getFetchOffset != fetchOffset)
      throw new RuntimeException("Offset doesn't match for partition [%s,%d] pti offset: %d fetch offset: %d"
                                .format(topicAndPartition.topic, topicAndPartition.partition, pti.getFetchOffset, fetchOffset))
    pti.enqueue(partitionData.messages.asInstanceOf[ByteBufferMessageSet])
  }
```

回想, ConsumerFetcherThread 是由 ConsumerFetcherManager 创建的, 也就是说它的 partitionMap 函数也是 Mgr 给出的,
ConsumerFetcherManager 又是由 ZookeeperConsumerConnector 确定的, 

```scala
ConsumerFetcherManager:  
  override def createFetcherThread(fetcherId: Int, sourceBroker: Broker): AbstractFetcherThread = {
    new ConsumerFetcherThread(
      "ConsumerFetcherThread-%s-%d-%d".format(consumerIdString, fetcherId, sourceBroker.id),
      config, sourceBroker, partitionMap, this)
  }

ZookeeperConsumerConnector:
  private def createFetcher() {
    if (enableFetcher)
      fetcher = Some(new ConsumerFetcherManager(consumerIdString, config, zkClient))
  }

```

走到这里, 基本上可以看出 Kafka Consumer 的逻辑了, ZookeeperConsumerConnector 是一个消费者进程, 管理获取消息的
所有信息, 而 ConsumerFetcherManager 是负责从 broker 中获取 log 信息, 它维持一个 (partition, partitionData) 的
映射, 里面包含最新的 Message

ZookeeperConsumerConnector 负责 partition, consumer, broker 的动态增加或减少, 它影响的方式是 AbstractFetcherManager
中的 (BrokerAndFetcherId, AbstractFetcherThread), 当 partition, broker 变化时, AbstractFetcherThreads 都会跟着发生
变化.

那么放到 ConsumerFetcherManager 中的数据何时被消费者使用呢?


## to KafkaApis and Cluster

![](/images/posts/kafka/fetchRequestProcess.png)

```scala
KafkaApis:

def handleFetchRequest(request: RequestChannel.Request) {
    val fetchRequest = request.requestObj.asInstanceOf[FetchRequest]
    val dataRead = replicaManager.readMessageSets(fetchRequest)


ReplicaManager: 
  /**
   * Read from all the offset details given and return a map of
   * (topic, partition) -> PartitionData
   */
def readMessageSets(fetchRequest: FetchRequest) = {
  val isFetchFromFollower = fetchRequest.isFromFollower
  fetchRequest.requestInfo.map
    case (TopicAndPartition(topic, partition), PartitionFetchInfo(offset, fetchSize)) =>
      val partitionDataAndOffsetInfo =
      val (fetchInfo, highWatermark) = readMessageSet(topic, partition, offset, fetchSize, 
        fetchRequest.replicaId)
      new PartitionDataAndOffset(new FetchResponsePartitionData(ErrorMapping.NoError, 
        highWatermark, fetchInfo.messageSet), fetchInfo.fetchOffset)
    (TopicAndPartition(topic, partition), partitionDataAndOffsetInfo)


/**
 * Read from a single topic/partition at the given offset upto maxSize bytes
 */
private def readMessageSet(topic: String, partition: Int, offset: Long,
                             maxSize: Int, fromReplicaId: Int): (FetchDataInfo, Long) = {
    
    // check if the current broker is the leader for the partitions
    // parition.getReplica(ReplicaId)
    val localReplica: Replica = if(fromReplicaId == Request.DebuggingConsumerId)
        getReplicaOrException(topic, partition)
    else
        getLeaderReplicaIfLocal(topic, partition)
    
    val maxOffsetOpt =
        if (Request.isValidBrokerId(fromReplicaId)) None
        else Some(localReplica.highWatermark.messageOffset)
    
    val fetchInfo = localReplica.log match
        case Some(log) =>
            log.read(offset, maxSize, maxOffsetOpt)
        case None =>
            FetchDataInfo(LogOffsetMetadata.UnknownOffsetMetadata, MessageSet.Empty)
        
    (fetchInfo, localReplica.highWatermark.messageOffset)
```

从上面的代码可以看出, 请求已经到达了 Replica, 也就是 broker, 需要从 broker 的 log 中拿数据了

```scala
class Replica(val brokerId: Int,
              val partition: Partition,
              time: Time = SystemTime,
              initialHighWatermarkValue: Long = 0L,
              val log: Option[Log] = None)

class Log(val dir: File, //java.io.File
          @volatile var config: LogConfig,
          @volatile var recoveryPoint: Long = 0L,
          scheduler: Scheduler,
          time: Time = SystemTime) {

  private val segments: ConcurrentNavigableMap[java.lang.Long, LogSegment] = 
    new ConcurrentSkipListMap[java.lang.Long, LogSegment]         
```

其他相关的类

```scala
private[kafka] class Cluster {
    private val brokers = new mutable.HashMap[Int, Broker]
}
    
case class Broker(id: Int, host: String, port: Int)

class Partition(val topic: String,
                val partitionId: Int,
                time: Time,
                replicaManager: ReplicaManager) {
  
  private val localBrokerId = replicaManager.config.brokerId
  private val logManager = replicaManager.logManager
  private val zkClient = replicaManager.zkClient
  private val assignedReplicaMap = new Pool[Int, Replica]

class ReplicaManager(val config: KafkaConfig, 
                     time: Time,
                     val zkClient: ZkClient,
                     scheduler: Scheduler,
                     val logManager: LogManager,
                     val isShuttingDown: AtomicBoolean) {
  private val localBrokerId = config.brokerId
  private val allPartitions = new Pool[(String, Int), Partition]
  val replicaFetcherManager = new ReplicaFetcherManager(config, this)
```

深入到 log 具体怎么存的起码还得写两篇 1000 行的文章, 暂时跳过底层细节, 继续前面的 readMessageSet,
log.read 代码

问题: 客户端fetch数据,是按照Partition级别指定startOffset的,而一个Partition有多个Segment.

如果fetch的数据跨越多个Segment,怎么确保返回多个Segment的数据给客户端? 似乎下面的代码针对的是一个Segment.


```scala
def read(startOffset: Long, maxLength: Int, maxOffset: Option[Long] = None): FetchDataInfo = {
  var entry = segments.floorEntry(startOffset) // 小于startOffset的最大的那一条
  while(entry != null) {
    // logSegment.read
    val fetchInfo = entry.getValue.read(startOffset, maxOffset, maxLength, entry.getValue.size)
    if(fetchInfo == null) entry = segments.higherEntry(entry.getKey)  // 大于指定key的最小那一条
    else return fetchInfo
  }
}
```

![](/images/posts/kafka/logsegment_read.png)

```scala
private[log] def translateOffset(offset: Long, startingFilePosition: Int = 0): OffsetPosition = {
  val mapping = index.lookup(offset)  //返回值是绝对offset和其物理位置
  log.searchFor(offset, max(mapping.position, startingFilePosition))
}
def read(startOffset: Long, maxOffset: Option[Long], maxSize: Int, maxPosition: Long = size): FetchDataInfo = {
  val logSize = log.sizeInBytes         // this may change, need to save a consistent copy
  val startPosition = translateOffset(startOffset)
  if(startPosition == null) return null // if the start position is already off the end of the log, return null
  val offsetMetadata = new LogOffsetMetadata(startOffset, this.baseOffset, startPosition.position)

  // calculate the length of the message set to read based on whether or not they gave us a maxOffset
  val length = maxOffset match {
    case None => min((maxPosition - startPosition.position).toInt, maxSize)  // no max offset, just read until the max position
    case Some(offset) => {
        // there is a max offset, translate it to a file position and use that to calculate the max read size
        if(offset >= startOffset) {
          val mapping = translateOffset(offset, startPosition.position)
          // the max offset is off the end of the log, use the end of the file
          val endPosition = if(mapping == null) logSize else mapping.position
          min(min(maxPosition, endPosition) - startPosition.position, maxSize).toInt
        }
    }
  }
  FetchDataInfo(offsetMetadata, log.read(startPosition.position, length))
}
```

上面有两处translateOffset,第一次是对startOffset获取startPosition,第二次是对maxOffset获取endPosition.

maxOffset通常是Replica的HW,即消费者最多只能读取到hw这个位置的消息,当然前提是maxOffset要大于startOffset.

![](/images/posts/kafka/FileMessageSetProcess.png)

PartitionTopicInfo除了fetchedOffset用于抓取数据,还有一个consumedOffset.
而且ZookeeperConsumerConnector中除了fetchOffsets也有commitOffsets的操作.

fetchOffsets会指示消费者从Partition的起始位置开始抓取数据, 有了fetchOffset,配合抓取线程,以及fetch动作,返回messageSet
由于客户端要记录自己消费过的offset,所以使用consumedOffset

fetch的返回值虽然含有messageSet,但是这仅仅是服务端返回的一个视图对象
客户端要自己去迭代视图对象里的消息,才能真正获取到messageSet中的每条消息
所以这就是为什么ConsumerFetcherThread在fetch之后要有processPartitionData
PartitionData包括了messageSet视图对象,处理PartitionData首先放入队列中
然后客户端通过队列和KafkaStream,迭代获取消息!


### From KafkaStream to internals

KafkaStream 是 kafka 与外部 client 的接口, 从 KafkaStream 到内部的实现

```scala
class KafkaStream[K,V](private val queue: BlockingQueue[FetchedDataChunk],
                        consumerTimeoutMs: Int,
                        private val keyDecoder: Decoder[K],
                        private val valueDecoder: Decoder[V],
                        val clientId: String)
   extends Iterable[MessageAndMetadata[K,V]] with java.lang.Iterable[MessageAndMetadata[K,V]] {

  private val iter: ConsumerIterator[K,V] =
    new ConsumerIterator[K,V](queue, consumerTimeoutMs, keyDecoder, valueDecoder, clientId)
```

KafkaStream 是一个装饰则模式, 它的内部实现是 ConsumerIterator. 但从 KafkaStream 本身也可以看出, 它内部
有个 BlockingQueue, 从 kafka broker 拿到的数据全部都存在这个 Queue 里面

### 创建 KafkaStream

KafkaStream 是由 ZookeeperConsumerConnector 创建的

```scala
// (Topic, Threadid, BlockingQueue)
val topicThreadIdAndQueues = new Pool[(String, ConsumerThreadId), BlockingQueue[FetchedDataChunk]]

def consume[K, V](topicCountMap: scala.collection.Map[String, Int], keyDecoder: Decoder[K], valueDecoder: Decoder[V])
  : Map[String, List[KafkaStream[K, V]]] = {
 
  val topicCount = TopicCount.constructTopicCount(consumerIdString, topicCountMap)
  val topicThreadIds = topicCount.getConsumerThreadIdsPerTopic
  
  val queuesAndStreams = topicThreadIds.values.map(threadIdSet =>
    threadIdSet.map(_ => {
      val queue = new LinkedBlockingQueue[FetchedDataChunk](config.queuedMaxMessages)
      val stream = new KafkaStream[K, V](
        queue, config.consumerTimeoutMs, keyDecoder, valueDecoder, config.clientId)
          (queue, stream)
      })).flatten.toList
      
  val dirs = new ZKGroupDirs(config.groupId)
  registerConsumerInZK(dirs, consumerIdString, topicCount)
  reinitializeConsumer(topicCount, queuesAndStreams) // <- queuesAndStreams
  loadBalancerListener.kafkaMessageAndMetadataStreams.asInstanceOf[Map[String, List[KafkaStream[K, V]]]]
    
reinitializeConsumer:
    threadQueueStreamPairs.foreach(e => {
          val topicThreadId = e._1
          val q = e._2._1
          topicThreadIdAndQueues.put(topicThreadId, q)
          debug("Adding topicThreadId %s and queue %s to topicThreadIdAndQueues data structure"
            .format(topicThreadId, q.toString))
        })
    
```

这样, kafka 的 stream 就和 BlockingQueue 连接起来了

不过, 我又忘了 ConsumerFetcherManager 或者 ConsumerFetcherThread 是怎么把数据连到 ZookeeperConsumerConnector 的
我记得获得的 fetcherThread 拿到的数据会保存到一个 ConsumerFetcherThread.partitionMap 中, 而 partitionMap
又是 ConsumerFetcherManager 创建好传给 fetcherThread 的,

那么 ConsumerFetcherManager 的信息数据是怎么传到 ZookeeperConsumerConnector? 没看到数据是怎么联系起来的, 就这样吧


### ConsumerIterator

我目前还搞不懂 ConsumerIterator 是针对一个 client 线程还是整个 consumerGroup 的

```scala
class ConsumerIterator[K, V](private val channel: BlockingQueue[FetchedDataChunk],
                             consumerTimeoutMs: Int,
                             private val keyDecoder: Decoder[K],
                             private val valueDecoder: Decoder[V],
                             val clientId: String)
  extends IteratorTemplate[MessageAndMetadata[K, V]] with Logging {
```

Consumer 是集成 IteratorTemplate, 正好借此机会认识一下 Iterator 的实现

Iterator 至少有两个方法, 第一个是 hasNext 第二个是 next, 每次调用 next 之前先查看 hasNext 的返回值是否为 true

hasNext 的内部实现是, 去查看内部的成员变量状态, 假如是 ready 的则返回 true, 如果是 done 返回 true, 如果未知,
则尝试获取数据, 并把获得的内部数据记录到成员变量上。假如直接调用 next 则会出现 noSuchElementException

```scala
abstract class IteratorTemplate[T] extends Iterator[T] with java.util.Iterator[T] {
    private var state: State = NOT_READY
    private var nextItem = null.asInstanceOf[T]
    
    def hasNext(): Boolean = {
        if(state == FAILED)
          throw new IllegalStateException("Iterator is in failed state")
        state match {
          case DONE => false
          case READY => true
          case _ => maybeComputeNext()}}
    
    protected def makeNext(): T // 延迟到子类实现
    def maybeComputeNext(): Boolean = {
        state = FAILED
        nextItem = makeNext()
        if(state == DONE) {
          false
        } else {
          state = READY
          true}}
    
    def next(): T = {
        if(!hasNext())
          throw new NoSuchElementException()
        state = NOT_READY
        if(nextItem == null)
          throw new IllegalStateException("Expected item but none found.")
        nextItem}
```

ConsumerIterator 对 makeNext 实现

```scala

private var current: AtomicReference[Iterator[MessageAndOffset]] = new AtomicReference(null)
private var currentTopicInfo: PartitionTopicInfo = null

case class FetchedDataChunk(messages: ByteBufferMessageSet,
                            topicInfo: PartitionTopicInfo,
                            fetchOffset: Long)

// 具体到一个 partition 的, 所以一个 ConsumerIterator 已经也是具体到一个 Partition 的
class PartitionTopicInfo(val topic: String,
                         val partitionId: Int,
                         private val chunkQueue: BlockingQueue[FetchedDataChunk],
                         private val consumedOffset: AtomicLong,
                         private val fetchedOffset: AtomicLong,
                         private val fetchSize: AtomicInteger,
                         private val clientId: String) extends Logging {

protected def makeNext(): MessageAndMetadata[K, V] = {
    var currentDataChunk: FetchedDataChunk = null
    var localCurrent = current.get()
    if(localCurrent == null || !localCurrent.hasNext) {
        if (consumerTimeoutMs < 0) currentDataChunk = channel.take
        else // 超时时间的用法
            // 一次获取一个 DataChunk, 等于是有缓存的了
            currentDataChunk = channel.poll(consumerTimeoutMs, TimeUnit.MILLISECONDS)
            if (currentDataChunk == null) {
                resetState() // state = NOT_READY
                throw new ConsumerTimeoutException
            
            currentTopicInfo = currentDataChunk.topicInfo
            val cdcFetchOffset = currentDataChunk.fetchOffset
            val ctiConsumeOffset = currentTopicInfo.getConsumeOffset
            // message 怎么还有 iterator? def messages: ByteBufferMessageSet
            localCurrent = currentDataChunk.messages.iterator
            current.set(localCurrent)
    
    var item = localCurrent.next()
    
    // reject the messages that have already been consumed
    while (item.offset < currentTopicInfo.getConsumeOffset && localCurrent.hasNext) 
        item = localCurrent.next()
    consumedOffset = item.nextOffset
    new MessageAndMetadata(currentTopicInfo.topic, currentTopicInfo.partitionId, 
        item.message, item.offset, keyDecoder, valueDecoder)
```

上面的 while 循环有点看不懂, 每个 currentDataChunk 和它内部的 TopicInfo 有什么区别? 可能是 currentTopicInfo
表示当前线程拿到的数据, 而 chunkedData 包含的数据更多一些, 所以要 reject already consumed message



