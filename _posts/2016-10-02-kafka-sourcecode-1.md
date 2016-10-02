## Producer

### Usage

```java

case class KeyedMessage[K, V](val topic: String, val key: K, val partKey: Any, val message: V)

Properties props = new Properties();
props.put("metadata.broker.list", "broker1:9092,broker2:9092");
props.put("serializer.class", "kafka.serializer.StringEncoder");
props.put("partitioner.class", "example.producer.SimplePartitioner");
props.put("request.required.acks", "1");
ProducerConfig config = new ProducerConfig(props);
```

The first property, “metadata.broker.list” defines where the Producer can find a one or more 
Brokers to determine the Leader for each topic. This does not need to be the full set of Brokers 
in your cluster but should include at least two in case the first Broker is not available. 
No need to worry about figuring out which Broker is the leader for the topic (and partition), 
the Producer knows how to connect to the Broker and ask for the meta data then connect to the correct Broker.

The second property “serializer.class” defines what Serializer to use when preparing the message for 
transmission to the Broker. In our example we use a simple String encoder provided as part of Kafka. 
Note that the encoder must accept the same type as defined in the KeyedMessage object in the next step.

The third property  "partitioner.class" defines what class to use to determine which Partition 
in the Topic the message is to be sent to. This is optional, but for any non-trivial implementation 
you are going to want to implement a partitioning scheme. More about the implementation of this class later.
If you include a value for the key but haven't defined a partitioner.class Kafka will use the default 
partitioner. If the key is null, then the Producer will assign the message to a random Partition.

The last property "request.required.acks" tells Kafka that you want your Producer to require 
an acknowledgement from the Broker that the message was received. Without this setting the Producer 
will 'fire and forget' possibly leading to data loss. Additional information can be found here

```java
Producer<String, String> producer = new Producer<String, String>(config);

Random rnd = new Random();
long runtime = new Date().getTime();
String ip = “192.168.2.” + rnd.nextInt(255);
String msg = runtime + “,www.example.com,” + ip;

KeyedMessage<String, String> data = new KeyedMessage<String, String>("page_visits", ip, msg);
producer.send(data);
```

**自定义 partition**

```java
public class SimplePartitioner implements Partitioner {
    public SimplePartitioner (VerifiableProperties props) {
 
    }
    public int partition(Object key, int a_numPartitions) {
        int partition = 0;
        String stringKey = (String) key;
        int offset = stringKey.lastIndexOf('.');
        if (offset > 0) {
           partition = Integer.parseInt( stringKey.substring(offset+1)) % a_numPartitions;
        }
       return partition;
  }
}
```

使用命令行创建 topic, 并确认数据被写入 kafka

```bash
bin/kafka-create-topic.sh --topic page_visits --replica 3 --zookeeper localhost:2181 --partition 5

bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic page_visits --from-beginning
```

### 实现

Kafka提供了Producer类作为Java producer的api，该类有 sync 和 async 两种发送方式

Kafka中Producer异步发送消息是基于同步发送消息的接口来实现的，异步发送消息的实现很简单，客户端消息发送过来以后，
先放入到一个队列中然后就返回了。Producer再开启一个线程（ProducerSendThread）不断从队列中取出消息，
然后调用同步发送消息的接口将消息发送给Broker。

![](/images/posts/kafka/producer_workflow.png)

**创建Producer:** 

当new Producer(new ProducerConfig()),其底层实现，实际会产生两个核心类的实例：
Producer、DefaultEventHandler。在创建的同时，会默认new一个ProducerPool，即我们每new一个java的Producer类，
就会有创建Producer、EventHandler和ProducerPool，ProducerPool为连接不同kafka broker的池，
初始连接个数有broker.list参数决定

**调用producer.send方法流程:**

当应用程序调用 producer.send 方法时，其内部其实调的是 eventhandler.handle(message) 方法,eventHandler 会首先序列化
该消息

```java
eventHandler.serialize(events)-->dispatchSerializedData()-->partitionAndCollate()-->send()-->SyncProducer.send()
```

当客户端应用程序调用producer发送消息messages时(既可以发送单条消息，也可以发送List多条消息)，
调用eventhandler.serialize首先序列化所有消息，序列化操作用户可以自定义实现Encoder接口，
下一步调用partitionAndCollate根据topics的messages进行分组操作，messages分配给dataPerBroker(多个不同的Broker的Map)，
根据不同Broker调用不同的SyncProducer.send批量发送消息数据，SyncProducer包装了nio网络操作信息。

**partitionAndCollate方法详细作用:**

获取所有partitions的leader所在leaderBrokerId(就是在该 partitionId 的 leader 分布在哪个 broker 上),
创建一个HashMap<int, Map<TopicAndPartition, List<KeyedMessage<K,Message>>>>, 把messages按照brokerId分组
组装数据，然后为SyncProducer分别发送消息作准备工作

### Producer平滑扩容机制

如果开发过producer客户端代码，会知道metadata.broker.list参数，它的含义是 kafka broker 的 ip 和 port 列表，
producer初始化时，就连接这几个broker，这时大家会有疑问，producer支持 kafka cluster 新增 broker 节点？
它又没有监听zk broker节点或从zk中获取broker信息，答案是肯定的，producer 可以支持平滑扩容 broker，
他是通过定时与现有的metadata.broker.list通信，获取新增broker信息，然后把新建的 SyncProducer 放入ProducerPool中。
等待后续应用程序调用

DefaultEventHandler类中初始化实例化BrokerPartitionInfo类，然后定期brokerPartitionInfo.updateInfo方法，
DefaultEventHandler部分代码如下:

```java
def handle(events: Seq[KeyedMessage[K,V]]) {
    val serializedData = serialize(events)
    serializedData.foreach {
        keyed =>
            val dataSize = keyed.message.payloadSize
    }
    
    var outstandingProduceRequests = serializedData
    var remainingRetries = config.messageSendMaxRetries + 1
    while (remainingRetries > 0 && outstandingProduceRequests.size > 0) {
        topicMetadataToRefresh ++= outstandingProduceRequests.map(_.topic)
        if (topicMetadataRefreshInterval >= 0 &&
            SystemTime.milliseconds - lastTopicMetadataRefreshTime > topicMetadataRefreshInterval) {
        
            Utils.swallowError(brokerPartitionInfo.updateInfo(topicMetadataToRefresh.toSet, correlationId.getAndIncrement))
            sendPartitionPerTopicCache.clear()
            topicMetadataToRefresh.clear
            lastTopicMetadataRefreshTime = SystemTime.milliseconds
        }
        outstandingProduceRequests = dispatchSerializedData(outstandingProduceRequests)
        if (outstandingProduceRequests.size > 0) {
            Thread.sleep(config.retryBackoffMs)
            // get topics of the outstanding produce requests and refresh metadata for those
            Utils.swallowError(brokerPartitionInfo.updateInfo(outstandingProduceRequests.map(_.topic).toSet, correlationId.getAndIncrement))
            sendPartitionPerTopicCache.clear()
            remainingRetries -= 1
            producerStats.resendRate.mark()
        }
        
    if(outstandingProduceRequests.size > 0) {
        producerStats.failedSendRate.mark()
        val correlationIdEnd = correlationId.get()
        throw new FailedToSendMessageException("Failed to send messages after " + config.messageSendMaxRetries + " tries.", null)
    }
```

```java
  def updateInfo(topics: Set[String], correlationId: Int) {
    
    var topicsMetadata: Seq[TopicMetadata] = Nil
    
    //根据topics列表,meta.broker.list,其他配置参数,correlationId表示请求次数，一个计数器参数而已
    //创建一个topicMetadataRequest，并随机的选取传入的broker信息中任何一个去取metadata，直到取到为止
    val topicMetadataResponse = ClientUtils.fetchTopicMetadata(topics, brokers, producerConfig, correlationId)
    topicsMetadata = topicMetadataResponse.topicsMetadata
    // throw partition specific exception
    
    topicsMetadata.foreach(tmd =>{
      trace("Metadata for topic %s is %s".format(tmd.topic, tmd))
      if(tmd.errorCode == ErrorMapping.NoError) {
        topicPartitionInfo.put(tmd.topic, tmd)
      } else warn("Error while fetching metadata [%s] for topic [%s]: %s ".format(tmd, tmd.topic, ErrorMapping.exceptionFor(tmd.errorCode).getClass))
      
      tmd.partitionsMetadata.foreach(pmd =>{
        if (pmd.errorCode != ErrorMapping.NoError && pmd.errorCode == ErrorMapping.LeaderNotAvailableCode) {
          warn("Error while fetching metadata %s for topic partition [%s,%d]: [%s]".format(pmd, tmd.topic, pmd.partitionId,
            ErrorMapping.exceptionFor(pmd.errorCode).getClass))
        } // any other error code (e.g. ReplicaNotAvailable) can be ignored since the producer does not need to access the replica and isr metadata
      })
    })
    producerPool.updateProducer(topicsMetadata)
  }
```

```java
  def fetchTopicMetadata(topics: Set[String], brokers: Seq[Broker], producerConfig: ProducerConfig, correlationId: Int): TopicMetadataResponse = {
    var fetchMetaDataSucceeded: Boolean = false
    var i: Int = 0
    val topicMetadataRequest = new TopicMetadataRequest(TopicMetadataRequest.CurrentVersion, correlationId, producerConfig.clientId, topics.toSeq)
    var topicMetadataResponse: TopicMetadataResponse = null
    var t: Throwable = null
    val shuffledBrokers = Random.shuffle(brokers) //生成随机数
    while(i < shuffledBrokers.size && !fetchMetaDataSucceeded) {
      //对随机选到的broker会创建一个SyncProducer
      val producer: SyncProducer = ProducerPool.createSyncProducer(producerConfig, shuffledBrokers(i))
      info("Fetching metadata from broker %s with correlation id %d for %d topic(s) %s".format(shuffledBrokers(i), correlationId, topics.size, topics))
      try {  //发送topicMetadataRequest到该broker去取metadata，获得该topic所对应的所有的broker信息
        topicMetadataResponse = producer.send(topicMetadataRequest)
        fetchMetaDataSucceeded = true
      }
      catch {
        ......
      }
    }
    if(!fetchMetaDataSucceeded) {
      throw new KafkaException("fetching topic metadata for topics [%s] from broker [%s] failed".format(topics, shuffledBrokers), t)
    } else {
      debug("Successfully fetched metadata for %d topic(s) %s".format(topics.size, topics))
    }
    return topicMetadataResponse
  }
```

**ProducerPool的updateProducer**

```java

def updateProducer(topicMetadata: Seq[TopicMetadata]) {
    val newBrokers = new collection.mutable.HashSet[Broker]
    topicMetadata.foreach(tmd => {
      tmd.partitionsMetadata.foreach(pmd => {
        if(pmd.leader.isDefined)
          newBrokers+=(pmd.leader.get)
      })
    })
    lock synchronized {
      newBrokers.foreach(b => {
        if(syncProducers.contains(b.id)){
          syncProducers(b.id).close()
          syncProducers.put(b.id, ProducerPool.createSyncProducer(config, b))
        } else
          syncProducers.put(b.id, ProducerPool.createSyncProducer(config, b))
      })
    }
  }
```

笔者也是经常很长时间看源码分析，才明白了为什么 ProducerConfig 配置信息里面并不要求使用者提供完整的kafka集群的broker信息，
而是任选一个或几个即可。因为他会通过您选择的broker和topics信息而获取最新的所有的broker信息

值得了解的是用于发送TopicMetadataRequest的SyncProducer虽然是用ProducerPool.createSyncProducer方法建出来的，但用完并不还回ProducerPool，而是直接Close.

刷新metadata并不仅在第一次初始化时做。为了能适应kafka broker运行中因为各种原因挂掉、paritition改变等变化，
eventHandler会定期的再去刷新一次该metadata，刷新的间隔用参数topic.metadata.refresh.interval.ms定义，默认值是10分钟。

这里有三点需要强调：

1. 客户端调用send, 才会新建SyncProducer，只有调用send才会去定期刷新metadata
2. 在每次取metadata时，kafka会新建一个SyncProducer去取metadata，逻辑处理完后再close。
3. 根据当前SyncProducer(一个Broker的连接)取得的最新的完整的metadata，刷新ProducerPool中到broker的连接.
4. 每10分钟的刷新会直接重新把到每个broker的socket连接重建，意味着在这之后的第一个请求会有几百毫秒的延迟。
   如果不想要该延迟，把topic.metadata.refresh.interval.ms值改为-1，这样只有在发送失败时，才会重新刷新。
   Kafka的集群中如果某个partition所在的broker挂了，可以检查错误后重启重新加入集群，手动做rebalance，
   producer的连接会再次断掉，直到 rebalance 完成，那么刷新后取到的连接着中就会有这个新加入的broker

说明：每个SyncProducer实例化对象会建立一个socket连接

在ClientUtils.fetchTopicMetadata调用完成后，回到 BrokerPartitionInfo.updateInfo继续执行，
在其末尾，pool会根据上面取得的最新的metadata建立所有的SyncProducer，即Socket通道
producerPool.updateProducer(topicsMetadata)

在ProducerPool中，SyncProducer的数目是由该topic的partition数目控制的，即每一个SyncProducer对应一个broker，
内部封了一个到该broker的socket连接。每次刷新时，会把已存在SyncProducer给close掉，即关闭socket连接，
然后新建SyncProducer，即新建socket连接，去覆盖老的。 如果不存在，则直接创建新的。

### ProducerPool, SyncProducer和BlockingChannel

它们在一起是完成最后的数据发送任务。先来看它们的类图：

![](/images/posts/kafka/producer_pool.png)

ProducerPool中有一个HashMap，其key为brokerid，value为连接到这个broker的SyncProducer。
因此ProducerPool的更准确名字应该为SyncProducerPool。

BlockingChannel可以看成是一个Socket客户端，它有两个成员变量分别是机器名和端口号。它的connect方法会打开到对应机器的socket。它的send方法可以发送RequestOrResponse，它是真正发送数据的地方。

SyncProducer提供了两个send方法，分别用来发送ProducerRequest和TopicMetadataRequest。它内部是调用了blockingChannel来发送数据的。



## Kafka 在 Zookeeper 中的存储结构

![](/images/posts/kafka/producer_pool.png)

### topic注册信息

/brokers/topics/[topic] : 存储某个topic的partitions所有分配信息

```json
Schema:
{
    "version": "版本编号目前固定为数字1",
    "partitions": {
        "partitionId编号": [
            同步副本组brokerId列表
        ],
        "partitionId编号": [
            同步副本组brokerId列表
        ],
        .......
    }
}
Example:
{
    "version": 1,
    "partitions": {
    "0": [1, 2], // 左边为patitions编号，右边为同步副本组brokerId列表
    "1": [2, 1],
    "2": [1, 2],
    }
}
```
### 2.partition状态信息

```
/brokers/topics/[topic]/partitions/[0...N]  其中[0..N]表示partition索引号

/brokers/topics/[topic]/partitions/[partitionId]/state
```

``` json
Schema:
{
    "controller_epoch": 表示kafka集群中的中央控制器选举次数,
    "leader": 表示该partition选举leader的brokerId,
    "version": 版本编号默认为1,
    "leader_epoch": 该partition leader选举次数,
    "isr": [同步副本组brokerId列表]
}
 
Example:
{
    "controller_epoch": 1,
    "leader": 2,
    "version": 1,
    "leader_epoch": 0,
    "isr": [2, 1]
}
```

### 3. Broker注册信息

/brokers/ids/[0...N]  每个broker的配置文件中都需要指定一个数字类型的id(全局不可重复),此节点为临时znode(EPHEMERAL)

```json
Schema:
{
    "jmx_port": jmx端口号,
    "timestamp": kafka broker初始启动时的时间戳,
    "host": 主机名或ip地址,
    "version": 版本编号默认为1,
    "port": kafka broker的服务端端口号,由server.properties中参数port确定
}
 
Example:
{
    "jmx_port": 6061,
    "timestamp":"1403061899859"
    "version": 1,
    "host": "192.168.1.148",
    "port": 9092
}
```
 

### 4. Controller epoch: 

/controller_epoch -> int (epoch)   

此值为一个数字,kafka集群中第一个broker第一次启动时为1，以后只要集群中center controller中央控制器所在broker变更或挂掉，就会重新选举新的center controller，每次center controller变更controller_epoch值就会 + 1; 

### 5. Controller注册信息:

/controller -> int (broker id of the controller)  存储center controller中央控制器所在kafka broker的信息

```json
Schema:
{
    "version": 版本编号默认为1,
    "brokerid": kafka集群中broker唯一编号,
    "timestamp": kafka broker中央控制器变更时的时间戳
}
 
Example:
{
    "version": 1,
    "brokerid": 3,
    "timestamp": "1403061802981"
}

```

### Consumer and Consumer group概念: 

![](/images/posts/kafka/consumer_group.png)

1. a.每个consumer客户端被创建时,会向zookeeper注册自己的信息;
2. b.此作用主要是为了"负载均衡".
3. c.同一个Consumer Group中的Consumers，Kafka将相应Topic中的每个消息只发送给其中一个Consumer
4. d.Consumer Group中的每个Consumer读取Topic的一个或多个Partitions，并且是唯一的Consumer
5. e.一个Consumer group的多个consumer的所有线程依次有序地消费一个topic的所有partitions,如果Consumer group中所有consumer总线程大于partitions数量，则会出现空闲情况;

**举例说明：**

kafka集群中创建一个topic为report-log   4 partitions 索引编号为0,1,2,3

假如有目前有三个消费者node：注意-->一个consumer中一个消费线程可以消费一个或多个partition.

如果每个consumer创建一个consumer thread线程,各个node消费情况如下，node1消费索引编号为0,1分区，node2费索引编号为2,node3费索引编号为3

如果每个consumer创建2个consumer thread线程，各个node消费情况如下(是从consumer node先后启动状态来确定的)，node1消费索引编号为0,1分区；node2费索引编号为2,3；node3为空闲状态

**总结：**

从以上可知，Consumer Group中各个consumer是根据先后启动的顺序有序消费一个topic的所有partitions的。
如果Consumer Group中所有consumer的总线程数大于partitions数量，则可能consumer thread或consumer会出现空闲状态。

**Consumer均衡算法**

当一个group中,有consumer加入或者离开时,会触发partitions均衡.均衡的最终目的,是提升topic的并发消费能力.

1) 假如topic1,具有如下partitions: P0,P1,P2,P3

2) 加入group中,有如下consumer: C0,C1

3) 首先根据partition索引号对partitions排序: P0,P1,P2,P3

4) 根据(consumer.id + '-'+ thread序号)排序: C0,C1

5) 计算倍数: M = [P0,P1,P2,P3].size / [C0,C1].size,本例值M=2(向上取整)

6) 然后依次分配partitions: C0 = [P0,P1],C1=[P2,P3],即Ci = [P(i * M),P((i + 1) * M -1)]

## Consumer注册信息:

每个consumer都有一个唯一的ID(consumerId可以通过配置文件指定,也可以由系统生成),此id用来标记消费者信息.

/consumers/[groupId]/ids/[consumerIdString]

是一个临时的znode,此节点的值为请看consumerIdString产生规则,即表示此consumer目前所消费的topic + partitions列表

consumerId产生规则:

```java
   StringconsumerUuid = null;
    if(config.consumerId!=null && config.consumerId)
      consumerUuid = consumerId;
    else {
      String uuid = UUID.randomUUID()
      consumerUuid = "%s-%d-%s".format(
        InetAddress.getLocalHost.getHostName, System.currentTimeMillis,
        uuid.getMostSignificantBits().toHexString.substring(0,8));
     }
     String consumerIdString = config.groupId + "_" + consumerUuid;
```

```
Schema:
{
    "version": 版本编号默认为1,
    "subscription": { //订阅topic列表
    "topic名称": consumer中topic消费者线程数
    },
    "pattern": "static",
    "timestamp": "consumer启动时的时间戳"
}
 
Example:
{
    "version": 1,
    "subscription": {
    "open_platform_opt_push_plus1": 5
    },
    "pattern": "static",
    "timestamp": "1411294187842"
}
```

### Consumer owner:

/consumers/[groupId]/owners/[topic]/[partitionId] -> consumerIdString + threadId索引编号

当consumer启动时,所触发的操作:

1. a) 首先进行"Consumer Id注册";

2. b) 然后在"Consumer id 注册"节点下注册一个watch用来监听当前group中其他consumer的"退出"和"加入";只要此znode path下节点列表变更,都会触发此group下consumer的负载均衡.(比如一个consumer失效,那么其他consumer接管partitions).

3. c) 在"Broker id 注册"节点下,注册一个watch用来监听broker的存活情况;如果broker列表变更,将会触发所有的groups下的consumer重新balance.

### 8. Consumer offset:

/consumers/[groupId]/offsets/[topic]/[partitionId] -> long (offset)

用来跟踪每个consumer目前所消费的partition中最大的offset

此znode为持久节点,可以看出offset跟group_id有关,以表明当消费者组(consumer group)中一个消费者失效,

重新触发balance,其他consumer可以继续消费.

### 9. Re-assign partitions

/admin/reassign_partitions

```
{
   "fields":[
      {
         "name":"version",
         "type":"int",
         "doc":"version id"
      },
      {
         "name":"partitions",
         "type":{
            "type":"array",
            "items":{
               "fields":[
                  {
                     "name":"topic",
                     "type":"string",
                     "doc":"topic of the partition to be reassigned"
                  },
                  {
                     "name":"partition",
                     "type":"int",
                     "doc":"the partition to be reassigned"
                  },
                  {
                     "name":"replicas",
                     "type":"array",
                     "items":"int",
                     "doc":"a list of replica ids"
                  }
               ],
            }
            "doc":"an array of partitions to be reassigned to new replicas"
         }
      }
   ]
}
 
Example:
{
  "version": 1,
  "partitions":
     [
        {
            "topic": "Foo",
            "partition": 1,
            "replicas": [0, 1, 3]
        }
     ]            
}
```

### 10. Preferred replication election

/admin/preferred_replica_election

```
{
   "fields":[
      {
         "name":"version",
         "type":"int",
         "doc":"version id"
      },
      {
         "name":"partitions",
         "type":{
            "type":"array",
            "items":{
               "fields":[
                  {
                     "name":"topic",
                     "type":"string",
                     "doc":"topic of the partition for which preferred replica election should be triggered"
                  },
                  {
                     "name":"partition",
                     "type":"int",
                     "doc":"the partition for which preferred replica election should be triggered"
                  }
               ],
            }
            "doc":"an array of partitions for which preferred replica election should be triggered"
         }
      }
   ]
}
 
例子:
 
{
  "version": 1,
  "partitions":
     [
        {
            "topic": "Foo",
            "partition": 1         
        },
        {
            "topic": "Bar",
            "partition": 0         
        }
     ]            
}
```

### 11. 删除topics

/admin/delete_topics

```
Schema:
{ "fields":
    [ {"name": "version", "type": "int", "doc": "version id"},
      {"name": "topics",
       "type": { "type": "array", "items": "string", "doc": "an array of topics to be deleted"}
      } ]
}
 
例子:
{
  "version": 1,
  "topics": ["foo", "bar"]
}
```

### Topic配置

/config/topics/[topic_name]
例子

```
{
  "version": 1,
  "config": {
    "config.a": "x",
    "config.b": "y",
    ...
  }
}
```