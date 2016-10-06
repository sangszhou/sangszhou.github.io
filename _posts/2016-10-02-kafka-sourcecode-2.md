## Kafka 0.8.2 新的offset管理

之前Kafka存在的一个非常大的性能问题(隐患)就是利用ZK来记录各个Consumer Group的消费进度。当然JVM Client帮我们自动做了这些事情，
但是Consumer需要和ZK频繁交互，而利用ZK Client API对ZK频繁写入是一个低效的操作，并且从水平扩展性上来讲也存在问题。所以ZK抖一抖，
集群吞吐量就跟着一起抖，严重的时候简直抖的停不下来。 当然某些非JVM端的API压根就不把offset存在ZK中，如基于Go的sarama，
直接自己维护了一个变量在内存中记录，总之就是问题多多了。所以非JVM的客户端，大家悠着点用。
****
0.8.2 Kafka引入了一个叫native offset storage的玩意儿，将offset管理从ZK移出，并且可以做到水平扩展。看到这个消息大家基本都泪流满面了。

实现原理其实也很自然，利用了Kafka自己的compacted topic，以consumer group，topic与Partition的组合作为key。
所以offset的提交就直接写到上述说的compacted topic中了，但是又由于offset是如此的重要以至于绝逼不能丢数据，
所以消息的acking级别(由request.required.acks控制)设置为-1，producer等到所有ISR收到消息后才会得到ack(数据安全性最好，但是速度会有影响)。
所以Kafka又在内存中维护了<consumer group,topic,partition>的三元组来维护最新的offset信息，consumer来取最新offset信息的时候直接内存里拿即可。
当然，敏锐性强的朋友一定想到了，kafka允许你快速的checkpoint最新的offset信息到磁盘上。

## Kafka 项目结构

**admin**      --管理员模块，操作和管理topic，paritions相关，包含create,delete topic,扩展	             		patitions

**Api**       --该模块主要负责交互数据的组装，客户端与服务端交互数据编解码

**client**    --该模块比较简单，就一个类，Producer读取kafka broker元数据信息，topic和partitions，以及leader

**cluster**   --该模块包含几个实体类，Broker,Cluster,Partition,Replica,解释他们之间关系:

Cluster由多个broker组成，一个Broker包含多个partition，一个topic的所有partitions分布在不同broker的中，一个Replica包含多个Partition。

**common**     --通用模块,只包含异常类和错误验证

**consumer**    --consumer处理模块，负责所有客户端消费者数据和逻辑处理

**controller**  --负责中央控制器选举，partition的leader选举，副本分配，副本重新分配，partition和replica扩容。

**javaapi**      --提供java的producer和consumer接口api

**log**          --Kafka文件存储模块，负责读写所有kafka的topic消息数据。

**message**    --封装多个消息组成一个“消息集”或压缩消息集。

**metrics**    --内部状态的监控模块

**network**           --网络事件处理模块，负责处理和接收客户端连接

**producer**    --producer实现模块，包括同步和异步发送消息。

**serializer**  --序列化或反序列化当前消息

**kafka**           --kafka门面入口类，副本管理，topic配置管理，leader选举实现(由contoroller模块调用)。

**tools**           --一看这就是工具模块，包含内容比较多：

1. a. 导出对应consumer的offset值.
2. b. 导出LogSegments信息，当前topic的log写的位置信息.
3. c.导出zk上所有consumer的offset值.
4. d.修改注册在zk的consumer的offset值.
5. f.producer和consumer的使用例子.

**utils**		  --Json工具类，Zkutils工具类，Utils创建线程工具类，KafkaScheduler公共调度器类，公共日志类等等。


kafka中KafkaServer类，采用门面模式，是网络处理，io处理等得入口.

ReplicaManager    副本管理

KafkaApis    处理所有request的Proxy类,根据requestKey决定调⽤用具体的handler

KafkaRequestHandlerPool 处理request的线程池，请求处理池  <-- num.io.threads io线程数量

LogManager    kafka文件存储系统管理，负责处理和存储所有Kafka的topic的partiton数据

TopicConfigManager  监听此zk节点的⼦子节点/config/changes/,通过LogManager更新topic的配置信息，topic粒度配置管理，具体请查看topic级别配置

KafkaHealthcheck 监听zk session expire,在zk上创建broker信息,便于其他broker和consumer获取其信息

KafkaController  kafka集群中央控制器选举，leader选举，副本分配。

KafkaScheduler  负责副本管理和日志管理调度等等

ZkClient         负责注册zk相关信息.

BrokerTopicStats  topic信息统计和监控

ControllerStats          中央控制器统计和监控

### Kafka server startup process

```java
def startup() {  
    info("starting")  
    isShuttingDown = new AtomicBoolean(false)  
    shutdownLatch = new CountDownLatch(1)  
  
    /* start scheduler */  
    kafkaScheduler.startup()  
      
    /* setup zookeeper */  
    zkClient = initZk()  
  
    /* start log manager */  
    logManager = createLogManager(zkClient)  
    logManager.startup()  
  
    socketServer = new SocketServer(config.brokerId,  
                                    config.hostName,  
                                    config.port,  
                                    config.numNetworkThreads,  
                                    config.queuedMaxRequests,  
                                    config.socketSendBufferBytes,  
                                    config.socketReceiveBufferBytes,  
                                    config.socketRequestMaxBytes)  
    socketServer.startup()  
  
    replicaManager = new ReplicaManager(config, time, zkClient, kafkaScheduler, logManager, isShuttingDown)  
    kafkaController = new KafkaController(config, zkClient)  
      
    /* start processing requests */  
    apis = new KafkaApis(socketServer.requestChannel, replicaManager, zkClient, config.brokerId, config, kafkaController)  
    requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, config.numIoThreads)  
     
    Mx4jLoader.maybeLoad()  
  
    replicaManager.startup()  
  
    kafkaController.startup()  
      
    topicConfigManager = new TopicConfigManager(zkClient, logManager)  
    topicConfigManager.startup()  
      
    /* tell everyone we are alive */  
    kafkaHealthcheck = new KafkaHealthcheck(config.brokerId, config.advertisedHostName, config.advertisedPort, config.zkSessionTimeoutMs, zkClient)  
    kafkaHealthcheck.startup()  
  
      
    registerStats()  
    startupComplete.set(true);  
    info("started")  
  }  
    
  private def initZk(): ZkClient = {  
    info("Connecting to zookeeper on " + config.zkConnect)  
    val zkClient = new ZkClient(config.zkConnect, config.zkSessionTimeoutMs, config.zkConnectionTimeoutMs, ZKStringSerializer)  
    ZkUtils.setupCommonPaths(zkClient)  
    zkClient  
  }  
  
  /** 
   *  Forces some dynamic jmx beans to be registered on server startup. 
   */  
  private def registerStats() {  
    BrokerTopicStats.getBrokerAllTopicsStats()  
    ControllerStats.uncleanLeaderElectionRate  
    ControllerStats.leaderElectionTimer  
  }  
```

## Consumer 逻辑

### Consumer group 的注意事项

Consumer Group是整个Kafka集群全局的. 所以要特别小心在新的逻辑启动之前要关闭所有的旧的逻辑(消费者进程).

当新的消费者加入同一个消费组时,Kafka会添加这个消费者的线程到要消费的topic的可用线程集合中,并且触发re-balance.

在re-balance时,kafka会分配可用的partition给可用的线程,可能移动一个partition到其他的线程中.

如果你的消费者逻辑混合了新的和旧的处理逻辑,很可能有些消息会被分配到旧的处理逻辑中.

High Level Consumer可以(应该)是多线程的应用程序.线程模型是以topic的partitions数量为中心的,不过有些规则:

1. 如果线程数量多于partition的数量，有部分线程无法消费该topic下任何一条消息
2. 如果线程数量少于partition的数量，有一些线程会消费多个partition的数据
3. 如果线程数量等于partition的数量，则正好一个线程消费一个partition的数据
4. 当添加更多的消费者进程/线程会触发re-balance,导致partition的分配发生了变化

### 使用

单线程版本

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

  val streamIterator = consumer map (_.iterator())
  streamIterator foreach { ci: ConsumerIterator[Array[Byte], Array[Byte]] => {
  
    log.info("stream query foreach")
  
    if (System.currentTimeMillis() < requestEndTime && KafkaUtils.timedHasNext(ci)) {
  
        //if next throw exception, actor returns immediately
        val message = ci.next()
  
        val key = if(message.key == null) null else new String(message.key());
        val value = new String(message.message())
  
        log.debug(s"consumer actor get message: ($key, $value) for topic: $topic")
        lstbf += MessageContent(key, value)    }  
```

**多线程版本**

```java
public class ConsumerTest implements Runnable {
    private KafkaStream m_stream;
    private int m_threadNumber;
 
    public ConsumerTest(KafkaStream a_stream, int a_threadNumber) {
        m_threadNumber = a_threadNumber;
        m_stream = a_stream;
    }
 
    public void run() {
        ConsumerIterator<byte[], byte[]> it = m_stream.iterator();
        while (it.hasNext())
            System.out.println("Thread " + m_threadNumber + ": " + new String(it.next().message()));
        System.out.println("Shutting down Thread: " + m_threadNumber);
    }
}

public void run(int a_numThreads) {
    Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
    topicCountMap.put(topic, new Integer(a_numThreads));
    Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
    List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);
 
 
    // now launch all the threads
    //
    executor = Executors.newFixedThreadPool(a_numThreads);
 
    // now create an object to consume the messages
    //
    int threadNumber = 0;
    for (final KafkaStream stream : streams) {
        executor.execute(new ConsumerTest(stream, threadNumber)); // 开始执行
        threadNumber++;
    }
}
```

### Consumer

Consumer定义在ConsumerConnector接口同一个文件中.默认创建的ConsumerConnector是ZookeeperConsumerConnector

ConsumerConnector主要有创建消息流(createMessageStreams)和提交offset(commitOffsets)两种方法

Consumer会根据消息流消费数据, 并且定时提交offset.由客户端自己保存offset是kafka采用pull拉取消息的一个附带工作.

```scala
trait ConsumerConnector {
  def createMessageStreams(topicCountMap: Map[String,Int]): Map[String, List[KafkaStream[Array[Byte],Array[Byte]]]]
  def commitOffsets(offsetsToCommit: immutable.Map[TopicAndPartition, OffsetAndMetadata], retryOnFailure: Boolean)
  def setConsumerRebalanceListener(listener: ConsumerRebalanceListener)
  def shutdown()
}
```

一个Consumer会创建一个ZookeeperConsumerConnector,代表一个消费者进程.

```scala
private[kafka] class ZookeeperConsumerConnector(val config: ConsumerConfig, val enableFetcher: Boolean) 
        extends ConsumerConnector with Logging with KafkaMetricsGroup {
  private var fetcher: Option[ConsumerFetcherManager] = None
  private var zkUtils: ZkUtils = null
  private var topicRegistry = new Pool[String, Pool[Int, PartitionTopicInfo]]
  private val checkpointedZkOffsets = new Pool[TopicAndPartition, Long]
  private val topicThreadIdAndQueues = new Pool[(String, ConsumerThreadId), BlockingQueue[FetchedDataChunk]]
  private val scheduler = new KafkaScheduler(threads = 1, threadNamePrefix = "kafka-consumer-scheduler-")
  private val messageStreamCreated = new AtomicBoolean(false)
  private var offsetsChannel: BlockingChannel = null
  private var sessionExpirationListener: ZKSessionExpireListener = null
  private var topicPartitionChangeListener: ZKTopicPartitionChangeListener = null
  private var loadBalancerListener: ZKRebalancerListener = null
  private var wildcardTopicWatcher: ZookeeperTopicEventWatcher = null
  private var consumerRebalanceListener: ConsumerRebalanceListener = null

  connectZk()                       // ① 创建ZkUtils,会创建对应的ZkConnection和ZkClient
  createFetcher()                   // ② 创建ConsumerFetcherManager,消费者fetcher线程
  ensureOffsetManagerConnected()    // ③ 确保连接上OffsetManager.
  if (config.autoCommitEnable) {    // ④ 启动定时提交offset线程
    scheduler.startup              
    scheduler.schedule("kafka-consumer-autocommit", autoCommit, ...)
  }
}
```

**fetcher:** 消费者获取数据, 使用ConsumerFetcherManager fetcher线程抓取数据

**zkUtils:** 消费者要和ZK通信, 除了注册自己,还有其他信息也会写到ZK中

**topicThreadIdAndQueues:** 消费者会指定自己消费哪些topic,并指定线程数, 所以topicThreadId都对应一个队列

**messageStreamCreated:** 消费者会创建消息流, 每个队列都对应一个消息流

**offsetsChannel:** offset可以存储在ZK或者kafka中,如果存在kafka里,像其他请求一样,需要和Broker通信

还有其他几个Listener监听器,分别用于 topicPartition 的更新,负载均衡,消费者重新负载等

虽然high level consumer不需要在应用程序中自己管理offset等,但kafka内部还是会帮你管理offset的.

为什么ConsumerConnector是ZookeeperConsumerConnector,因为消费者的offset是保存在ZooKeeper中的.

broker 在 zookeeper 中的信息

```
[zk: 192.168.47.83:2181,192.168.47.84:2181,192.168.47.86:2181(CONNECTED) 1] ls /brokers
[topics, ids]

[zk: 192.168.47.83:2181,192.168.47.84:2181,192.168.47.86:2181(CONNECTED) 2] ls /brokers/ids
[3, 5, 4]
[zk: 192.168.47.83:2181,192.168.47.84:2181,192.168.47.86:2181(CONNECTED) 4] get /brokers/ids/3
{"jmx_port":10055,"timestamp":"1453380999577","host":"192.168.48.153","version":1,"port":9092}
```

```
[zk: 192.168.47.83:2181,192.168.47.84:2181,192.168.47.86:2181(CONNECTED) 17] get /brokers/topics/topic1
{"version":1,"partitions":{"2":[5,4],"1":[4,3],"0":[3,5]}}   ⬅️

[zk: 192.168.47.83:2181,192.168.47.84:2181,192.168.47.86:2181(CONNECTED) 12] ls /brokers/topics/topic1/partitions
[2, 1, 0]
[zk: 192.168.47.83:2181,192.168.47.84:2181,192.168.47.86:2181(CONNECTED) 13] ls /brokers/topics/topic1/partitions/0
[state]
[zk: 192.168.47.83:2181,192.168.47.84:2181,192.168.47.86:2181(CONNECTED) 15] get /brokers/topics/topic1/partitions/0/state
{"controller_epoch":1775,"leader":3,"version":1,"leader_epoch":145,"isr":[3,5]}
```

partition0 的 leader 是 3, 复制在 broker 3 和 5 中
 
**Consumer id registry:** /consumers/[group_id]/ids/[consumer_id] -> topic1,...topicN

每个消费者会将它的id注册为临时znode并且将它所消费的topic设置为znode的值,当客户端(消费者)退出时,znode(consumer_id)会被删除

**Partition owner registry:** /consumers/[group_id]/owner/[topic]/[broker_id-partition_id] --> consumer_node_id

在消费时,每个topic的partition只能被一个消费者组中的唯一的一个消费者消费.在每次重新负载的时候,这个映射策略就会重新构建

**Consumer offset tracking:** /consumers/[group_id]/offsets/[topic]/[broker_id-partition_id] --> offset_counter_value

每个消费者都要跟踪自己消费的每个Partition最近的offset.表示自己读取到Partition的最新位置.

由于一个Partition只能被消费组中的一个消费者消费,所以offset是以消费组为级别的,而不是消费者.

因为如果原来的消费者挂了后,应当将这个Partition交给同一个消费组中别的消费者,而此时offset是没有变化的.

一个partition可以被不同的消费者组中的不同消费者消费，所以不同的消费者组必须维护他们各自对该partition消费的最新的offset

### 初始化

1. 因为消费者要和ZK通信,所以connectZk会确保连接上ZooKeeper
2. 消费者要消费数据,需要有抓取线程,所有的抓取线程交给ConsumerFetcherManager统一管理
3. 由消费者客户端自己保存offset,而消费者会消费多个topic的多个partition.
4. 类似多个数据抓取线程有管理类,多个partition的offset管理类OffsetManager是一个GroupCoordinator
5. 定时提交线程会使用OffsetManager建立的通道定时提交offset到zk或者kafka.

### 消费

consume并不真正的消费数据,只是初始化存放数据的 queue.真正消费数据的是对该queue进行shallow iterator.

在kafka的运行过程中,会有其他的线程将数据放入partition对应的queue中. 而queue是用于KafkaStream的.

一旦数据添加到queue后,KafkaStream的阻塞队列就有数据了,消费者就可以从队列中消费消息.

```java
def createMessageStreams[K,V](topicCountMap: Map[String,Int], 
  keyDecoder: Decoder[K], valueDecoder: Decoder[V]) : Map[String, List[KafkaStream[K,V]]] = {
  consume(topicCountMap, keyDecoder, valueDecoder)
}

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

consumerIdString会返回当前Consumer在哪个ConsumerGroup的编号.每个consumer在消费组中的编号都是唯一的.

一个消费者,对一个topic可以使用多个线程一起消费(一个进程可以有多个线程). 当然一个消费者也可以消费多个topic.

```java
def makeConsumerThreadIdsPerTopic(consumerIdString: String, topicCountMap: Map[String,  Int]) = {
  val consumerThreadIdsPerTopicMap = new mutable.HashMap[String, Set[ConsumerThreadId]]()
 
  for ((topic, nConsumers) <- topicCountMap) {                // 每个topic有几个消费者线程
    val consumerSet = new mutable.HashSet[ConsumerThreadId]   // 一个消费者线程对应一个ConsumerThreadId
    
    for (i <- 0 until nConsumers)
      consumerSet += ConsumerThreadId(consumerIdString, i)
      consumerThreadIdsPerTopicMap.put(topic, consumerSet)      // 每个topic都有多个Consumer线程,但是只有一个消费者进程
  }
 
  consumerThreadIdsPerTopicMap                                // topic到消费者线程集合的映射
}
```

对于消费者而言,它只要指定要消费的topic和线程数量就可以了,其他具体这个topic分成多少个partition,
以及topic-partition是分布是哪个broker上,对于客户端而言都是透明的.
客户端关注的是我的每个线程都对应了一个队列,每个队列都是一个消息流就可以了.

在Producer以及前面分析的Fetcher,都是以Broker-Topic-Partition为级别的.
AbstractFetcherManager的fetcherThreadMap就是以brokerAndFetcherId来创建拉取线程的.
而消费者是通过拉取线程才有数据可以消费的,所以客户端的每个线程实际上也是针对Partition级别的.

### registerConsumerInZK

消费者需要向ZK注册一个临时节点,路径为:/consumers/[group_id]/ids/[consumer_id],内容为订阅的topic.

```java
private def registerConsumerInZK(dirs: ZKGroupDirs, consumerIdString: String, topicCount: TopicCount) {
  val consumerRegistrationInfo = Json.encode(Map("version" -> 1, 
    "subscription" -> topicCount.getTopicCountMap, "pattern" -> topicCount.pattern, "timestamp" -> timestamp))
  val zkWatchedEphemeral = new ZKCheckedEphemeral(dirs.consumerRegistryDir + "/" + consumerIdString, 
    consumerRegistrationInfo, zkUtils.zkConnection.getZookeeper, false)
  zkWatchedEphemeral.create()
}
```

当前Consumer在ZK注册之后,需要重新初始化Consumer. 对于全新的消费者,注册多个监听器,在zk的对应节点的注册事件发生时,会回调监听器的方法.

当前Consumer在ZK注册之后,需要重新初始化Consumer. 对于全新的消费者,注册多个监听器,在zk的对应节点的注册事件发生时,会回调监听器的方法.
将topic对应的消费者线程id及对应的LinkedBlockingQueue放入topicThreadIdAndQueues中,LinkedBlockingQueue是真正存放数据的queue

1. 注册sessionExpirationListener,监听状态变化事件.在session失效重新创建session时调用
2. 向/consumers/[group_id]/ids注册Child变更事件的loadBalancerListener,当消费组下的消费者发生变化时调用
3. 向/brokers/topics/[topic]注册Data变更事件的topicPartitionChangeListener,在topic数据发生变化时调用

显式调用loadBalancerListener.syncedRebalance(), 会调用reblance方法进行consumer的初始化工作

```java
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

ZKRebalancerListener传入ZKSessionExpireListener和ZKTopicPartitionChangeListener.它们都会使用ZKRebalancerListener完成自己的工作.

因为消费者加入/退出时,消费组的成员会发生变化,而消费组中的所有存活消费者负责消费可用的partitions.
可用的partitions或者消费组中的消费者成员一旦发生变化,都要重新分配partition给存活的消费者.下面是一个示例.

当然分配partition的工作绝不仅仅是这么简单的,还要处理与之相关的线程,并重建必要的数据:

1. 关闭数据抓取线程，获取之前为topic设置的存放数据的queue并清空该queue
2. 释放 partition的ownership, 删除 partition和 consumer 的对应关系
3. 为各个 partition 重新分配 threadId
   获取 partition 最新的 offset 并重新初始化新的 PartitionTopicInfo(queue存放数据,两个offset为partition最新的offset)
4. 重新将 partition 对应的新的 consumer 信息写入 zookeeper
5. 重新创建 partition 的 fetcher 线程

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
        true
    }
  }
}
```

rebalance操作涉及了以下内容:

1. PartitionOwnership: Partition的所有者(ownership)的删除和重建
2. AssignmentContext: 分配信息上下文
3. PartitionAssignor: 为Partition分配Consumer的算法
4. PartitionAssignment: Partition分配之后的上下文
5. PartitionTopicInfo: Partition的最终信息
6. Fetcher: 完成了rebalance,消费者就可以重新开始抓取数据

### PartitionOwnership

topicRegistry的数据结构是: topic -> (partition -> PartitionTopicInfo), 表示现有的topic注册信息.

当partition被consumer所拥有后, 会在zk中创建/consumers/[group_id]/owner/[topic]/[partition_id] --> consumer_node_id
释放所有partition的ownership, 数据来源于topicRegistry的topic-partition(消费者所属的group_id也是确定的).

所以deletePartitionOwnershipFromZK会删除/consumers/[group_id]/owner/[topic]/[partition_id]节点.

这样partition没有了owner,说明这个partition不会被consumer消费了,也就相当于consumer释放了partition.

```scala
private def releasePartitionOwnership(localTopicRegistry: Pool[String, Pool[Int, PartitionTopicInfo]])= {
  for ((topic, infos) <- localTopicRegistry) {
    for(partition <- infos.keys) {
      deletePartitionOwnershipFromZK(topic, partition)
    }
    localTopicRegistry.remove(topic)
  }
  allTopicsOwnedPartitionsCount = 0
}

private def deletePartitionOwnershipFromZK(topic: String, partition: Int) {
  val topicDirs = new ZKGroupTopicDirs(group, topic)
  val znode = topicDirs.consumerOwnerDir + "/" + partition
  zkUtils.deletePath(znode)
}
```

重建ownership. 参数partitionAssignment会指定partition(TopicAndPartition)要分配给哪个consumer(ConsumerThreadId)消费的.

```scala
private def reflectPartitionOwnershipDecision(partitionAssignment: Map[TopicAndPartition, ConsumerThreadId]): Boolean = {
  var successfullyOwnedPartitions : List[(String, Int)] = Nil
  val partitionOwnershipSuccessful = partitionAssignment.map { partitionOwner =>
    val topic = partitionOwner._1.topic
    val partition = partitionOwner._1.partition
    val consumerThreadId = partitionOwner._2

    // 返回/consumers/[group_id]/owner/[topic]/[partition_id]节点路径,然后创建节点,节点内容为:consumerThreadId
    val partitionOwnerPath = zkUtils.getConsumerPartitionOwnerPath(group, topic, partition)
    zkUtils.createEphemeralPathExpectConflict(partitionOwnerPath, consumerThreadId.toString)

    // 成功创建的节点,加入到列表中
    successfullyOwnedPartitions ::= (topic, partition)
    true
  }
  // 判断上面的创建节点操作(为consumer分配partition)是否有错误,一旦有一个有问题,就全部回滚(删除掉).只有所有成功才算成功
  val hasPartitionOwnershipFailed = partitionOwnershipSuccessful.foldLeft(0)((sum, decision) => sum + (if(decision) 0 else 1))
  if(hasPartitionOwnershipFailed > 0) {
    successfullyOwnedPartitions.foreach(topicAndPartition => deletePartitionOwnershipFromZK(topicAndPartition._1, topicAndPartition._2))
    false
  } else true
}
```

将可用的partitions以及消费者线程排序, 将partitions处于线程数,表示每个线程(不是消费者数量)平均可以分到几个partition.
如果除不尽,剩余的会分给前面几个消费者线程. 比如有两个消费者,每个都是两个线程,一共有5个可用的partitions:(p0-p4).
每个消费者线程(一共四个线程)可以获取到至少一共partition(5/4=1),剩余一个(5%4=1)partition分给第一个线程.

最后的分配结果为: p0 -> C1-0, p1 -> C1-0, p2 -> C1-1, p3 -> C2-0, p4 -> C2-1

![](http://img.blog.csdn.net/20160127083105757)

```scala
class RangeAssignor() extends PartitionAssignor with Logging {
  def assign(ctx: AssignmentContext) = {
    val valueFactory = (topic: String) => new mutable.HashMap[TopicAndPartition, ConsumerThreadId]
    // consumerThreadId -> (TopicAndPartition -> ConsumerThreadId). 所以上面的valueFactory的参数不对哦!
    val partitionAssignment = new Pool[String, mutable.Map[TopicAndPartition, ConsumerThreadId]](Some(valueFactory))
    for (topic <- ctx.myTopicThreadIds.keySet) {
      val curConsumers = ctx.consumersForTopic(topic)                   // 订阅了topic的消费者列表(4)
      val curPartitions: Seq[Int] = ctx.partitionsForTopic(topic)       // 属于topic的partitions(5)
      val nPartsPerConsumer = curPartitions.size / curConsumers.size    // 每个线程都可以有这些个partition(1)
      val nConsumersWithExtraPart = curPartitions.size % curConsumers.size  // 剩余的分给前面几个(1)
      for (consumerThreadId <- curConsumers) {                          // 每个消费者线程: C1_0,C1_1,C2_0,C2_1
        val myConsumerPosition = curConsumers.indexOf(consumerThreadId) // 线程在列表中的索引: 0,1,2,3
        val startPart = nPartsPerConsumer * myConsumerPosition + myConsumerPosition.min(nConsumersWithExtraPart)
        val nParts = nPartsPerConsumer + (if (myConsumerPosition + 1 > nConsumersWithExtraPart) 0 else 1)
        // Range-partition the sorted partitions to consumers for better locality. 
        // The first few consumers pick up an extra partition, if any.
        if (nParts > 0)
          for (i <- startPart until startPart + nParts) {
            val partition = curPartitions(i)
            // record the partition ownership decision 记录partition的ownership决定,即把partition分配给consumerThreadId
            val assignmentForConsumer = partitionAssignment.getAndMaybePut(consumerThreadId.consumer)
            assignmentForConsumer += (TopicAndPartition(topic, partition) -> consumerThreadId)
          }
        }
      }
    }
    // 前面的for循环中的consumers是和当前消费者有相同topic的,如果消费者的topic和当前消费者不一样,则在本次assign中不会为它分配的.  
    // assign Map.empty for the consumers which are not associated with topic partitions 没有和partition关联的consumer分配空的partiton.
    ctx.consumers.foreach(consumerId => partitionAssignment.getAndMaybePut(consumerId))
    partitionAssignment
  }
}
```

