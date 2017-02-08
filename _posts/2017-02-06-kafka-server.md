### Controller

为了减小Zookeeper的压力，同时也降低整个分布式系统的复杂度，Kafka引入了一个“中央控制器“，也就是Controller。 
其基本思路是：先通过Zookeeper在所有broker中选举出一个Controller，然后用这个Controller来控制其它所有的broker，而不是让zookeeper直接控制所有的机器。 
比如上面对/broker/ids的监听，并不是所有broker都监听此结点，而是只有Controller监听此结点，这样就把一个“分布式“问题转化成了“集中式“问题，即降低了Zookeeper负担，也便于控制逻辑的编写。

**topic与partition的增加／删除**

同样，作为1个分布式集群，当增加／删除一个topic或者partition的时候，不可能挨个通知集群的每1台机器。

这里的实现思路也是：管理端(Admin/TopicCommand)把增加/删除命令发送给Zk，Controller监听Zk获取更新消息, Controller再分发给相关的broker。

```java
//当某个session断了重连，就会调用这个监听器
public interface IZkStateListener {
    public void handleStateChanged(KeeperState state) throws Exception;

    public void handleNewSession() throws Exception;

    public void handleSessionEstablishmentError(final Throwable error) throws Exception;

}

//当某个结点的data变化之后（data变化，或者结点本事被删除）
public interface IZkDataListener {

    public void handleDataChange(String dataPath, Object data) throws Exception;

    public void handleDataDeleted(String dataPath) throws Exception;
}

//当某个结点的子结点发生变化
public interface IZkChildListener {
    public void handleChildChange(String parentPath, List<String> currentChilds) throws Exception;
}
```

当controller挂了，其它所有broker监听到此临时节点消失，然后争相创建此临时节点，谁创建成功，谁就成为新的Controller。

除了/controller节点，还有一个辅助的/controller_epoch，记录当前Controller的轮值数。

## KafkaController与ZookeeperLeaderElector

整个选举过程是通过这2个核心类实现的，其中ZookeeperLeaderElector是KafkaController的一个成员变量：


```java
//KafkaController的一个成员变量
  private val controllerElector = new ZookeeperLeaderElector(controllerContext, ZkUtils.ControllerPath, onControllerFailover,
    onControllerResignation, config.brokerId)
```

下图展示了选举的整个交互过程： 

1. KafkaController和ZookeeperLeaderElector内部各有1个Listener，一个监听session重连，1个监听/controller节点变化。 
2. 当session重连，或者/controller节点被删除，则调用elect()函数，发起重新选举。在重新选举之前，先判断自己是否旧的Controller，如果是，则先调用onResignation退位。 

```java
  def startup() = {
    inLock(controllerContext.controllerLock) {
      info("Controller starting up")
      registerSessionExpirationListener()   //第1种监听：SessionExpirationListener
      isRunning = true
      controllerElector.startup   //第2种监听：LeaderChangeListener
      info("Controller startup complete")
    }
  }

  class SessionExpirationListener() extends IZkStateListener with Logging {
    ...
    @throws(classOf[Exception])
    def handleNewSession() {
      info("ZK expired; shut down all controller components and try to re-elect")
      inLock(controllerContext.controllerLock) {
        onControllerResignation()   //先退位
        controllerElector.elect   //发起重新选举
      }
    }
    ...
  }

  class LeaderChangeListener extends IZkDataListener with Logging {

    @throws(classOf[Exception])
    def handleDataChange(dataPath: String, data: Object) {
      inLock(controllerContext.controllerLock) {
        val amILeaderBeforeDataChange = amILeader
        leaderId = KafkaController.parseControllerId(data.toString)

        if (amILeaderBeforeDataChange && !amILeader)
          onResigningAsLeader()  //自己以前是controller，现在不是，退位
      }
    }


    @throws(classOf[Exception])
    def handleDataDeleted(dataPath: String) {
      inLock(controllerContext.controllerLock) {
        debug("%s leader change listener fired for path %s to handle data deleted: trying to elect as a leader"
          .format(brokerId, dataPath))
        if(amILeader)
          onResigningAsLeader()  //关键点：controller死了，有可能不是因为自己死了。而是和zookeeper的session断了。但是自己还在。此时，自己先退休，再重新发起选举。
        elect   //发起重现选举
      }
    }
  }
```

## ReplicationManager

其中，ReplicaManager内部有一个成员变量，存储了该台机器上所有的partition, 然后对于每1个Partition，其内部又存储了其所有的replica，也就是ISR:
                                                 

```java
class ReplicaManager(val config: KafkaConfig,
                     metrics: Metrics,
                     time: Time,
                     jTime: JTime,
                     val zkUtils: ZkUtils,
                     scheduler: Scheduler,
                     val logManager: LogManager,
                     val isShuttingDown: AtomicBoolean,
                     threadNamePrefix: Option[String] = None) extends Logging with KafkaMetricsGroup {
  //核心变量：存储该节点上所有的Partition
  private val allPartitions = new Pool[(String, Int), Partition]


class Partition(val topic: String,
                val partitionId: Int,
                time: Time,
                replicaManager: ReplicaManager) extends Logging with KafkaMetricsGroup {
  private val localBrokerId = replicaManager.config.brokerId
  private val logManager = replicaManager.logManager
  private val zkUtils = replicaManager.zkUtils
  private val assignedReplicaMap = new Pool[Int, Replica]
  //核心变量：这个Partition的leader
  @volatile var leaderReplicaIdOpt: Option[Int] = None

  //核心变量：isr，也即除了leader以外，其它所有的活着的follower集合
  @volatile var inSyncReplicas: Set[Replica] = Set.empty[Replica]

class Replica(val brokerId: Int,
              val partition: Partition,
              time: Time = SystemTime,
              initialHighWatermarkValue: Long = 0L,
              val log: Option[Log] = None) extends Logging {

  。。。
  //核心变量：该Replica当前从leader那fetch消息的最近offset，简称为loe
  @volatile private[this] var logEndOffsetMetadata: LogOffsetMetadata = LogOffsetMetadata.UnknownOffsetMetadata
```

以partition t0p1为例，b2作为leader，b3, b5作为follower，整个过程如下： 

1. b2的SocketServer收到producer的ProduceRequest请求，把请求交给ReplicaManager处理, ReplicaManager调用自己的appendMessages函数，把消息存到本地日志。 
2. ReplicaManager生成一个DelayedProduce对象，放入DelayedProducerPurgatory中，等待followers来把该请求pull到自己机器上 
3. 2个followers会跟consumer一样，发送FetchRequest请求到SocketServer，ReplicaManager调用自己的fetchMessage函数返回日志，同时更新2个follower的LOE，并且判断DelayedProducer是否可以complete。如果可以，则发送ProduceResponse.

下面看一下FetchRequest的处理代码：

```java
//ReplicaManager
  def fetchMessages(timeout: Long,
                    replicaId: Int,
                    fetchMinBytes: Int,
                    fetchInfo: immutable.Map[TopicAndPartition, PartitionFetchInfo],
                    responseCallback: Map[TopicAndPartition, FetchResponsePartitionData] => Unit) {
    ...
    if(Request.isValidBrokerId(replicaId))  //关键性的判断：如果这个FetchRequest来自一个Replica，而不是普通的Consumer
      updateFollowerLogReadResults(replicaId, logReadResults) //check DelayedProducer是否可以complete了

//check相对应的DelayedProduce
  def tryCompleteDelayedProduce(key: DelayedOperationKey) {
    val completed = delayedProducePurgatory.checkAndComplete(key)
    debug("Request key %s unblocked %d producer requests.".format(key.keyLabel, completed))
  }
```

这里有3个关键点： 

1. 每个DelayedProduce，内部包含一个ProduceResponseCallback函数。当complete之后，该callback被回调，也就处理了ProduceRequest请求. 
1. leader处理ProduceRequest请求和followers同步日志，这2个事情是并行的。leader并不会等待2个followers同步完该消息，再处理下1个。 
3. 每一个ProduceRequest对于一个该请求写入日志时的requestOffset。判断该条消息同步是否完成，只要每个replica的Loe >= requestOffset就可以了，并不需要完全相等。

## TimeWheel and DelayedOperationPurgatory

```java
class ReplicaManager(val config: KafkaConfig,
                     metrics: Metrics,
                     time: Time,
                     jTime: JTime,
                     val zkUtils: ZkUtils,
                     scheduler: Scheduler,
                     val logManager: LogManager,
                     val isShuttingDown: AtomicBoolean,
                     threadNamePrefix: Option[String] = None) extends Logging with KafkaMetricsGroup {

  //关键组件：每来1个ProduceReuqest，写入本地日志之后。就会生成一个DelayedProduce对象，放入delayedProducePurgatory中。
  // 之后这个delayedProduce对象，要么在处理FetchRequest的时候，被complete()；要么在purgatory内部被超时.
  val delayedProducePurgatory = new DelayedOperationPurgatory[DelayedProduce](
    purgatoryName = "Produce", config.brokerId, config.producerPurgatoryPurgeIntervalRequests)

  val delayedFetchPurgatory = new DelayedOperationPurgatory[DelayedFetch](
    purgatoryName = "Fetch", config.brokerId, config.fetchPurgatoryPurgeIntervalRequests)
```

下图可以看出，DelayedProducePurgatory有2个核心部件，1个是一个Watchers的map，1个是Timer。对应的，DelayedProduce有2个角色：一个是DelayedOperation，同时它也是一个TimerTask。

每当处理一个ProduceRequest，就会生成一个DelayedProduce对象，被加入到一个Watchers中，同时它也会作为一个TimerTask，加入到Timer中。

最后这个DelayedProduce可能被接下来的Fetch满足，也可能在Timer中超时，给客户端返回超时错误。如果是前者，那就需要调用TimerTask.cancel，把该任务从Timer中删除。

```java
trait TimerTask extends Runnable {

  val expirationMs: Long // TimerTask的核心变量：过期时间（绝对时间）

  private[this] var timerTaskEntry: TimerTaskEntry = null

  def cancel(): Unit = {  //取消该TimerTask
    synchronized {
      if (timerTaskEntry != null) timerTaskEntry.remove()
      timerTaskEntry = null
    }
  }
```

**为什么需要TimingWheel**

关于DelayedQueue，在另一个序列JUC并发编程里面，已经讲过。其内部本质就是一个二根堆，所有Task按照过期时间排序，堆顶放置的就是最近要过期的元素。队列的入对／出对其复杂度都是Log(n)。

在Kafka的场景中，基于2根堆实现的定时器，有2个缺点： 

1. 在应对服务器大规模请求中，Log(n)的复杂度，还是不够高效 

2. 另1个更大的问题是，DelayedQueue不支持随机删除。而在Kafka的这个场景中，当一个DelayedProduce在过期之前被complete之后，就需要把其从Timer中删除。

Kakfa的作者也曾提到过，在旧的版本中，因为被complete的请求，不能及时从DelayedQueue删除，导致Queue吃完jvm的内存的情况。

所以Kafka的设计者，提供了基于TimingWheel的“定时器“实现，这个定时器有2个优点： 

1. Task的加入，取出，时间复杂度是O(1) 
2. 支持Task的随机删除

![](/images/posts/kafka/timewheel.png)

Timer是最外层类，表示一个定时器。其内部有一个TimingWheel对象，TimingWheel是有层次结构的，每个TimingWheel可能有parent TimingWheel（这个原理就类似我们生活中的水表，不同表盘有不同刻度）。

TimingWheel是一个时间刻度盘，每个刻度上有一个TimerTask的双向链表，称之为一个bucket。同1个bucket里面的所有Task，其过期时间相等。因此，每1个bucket有一个过期时间的字段。

除此之外，所有TimingWheel共用了一个DelayedQueue，这个DelayedQueue存储了所有的bucket，而不是所有的TimerTask。

Timer的3大核心功能

我们知道，对于一个Timer来说，有3大功能： 

1. 添加：把一个TimerTask加入Timer 
2. 过期：时间到了，执行所有那些过期的TimerTask 
3. 取消：时间未到，取消TimerTask。把TimerTask删除

一方面，调用者（也就是DelayedOperationPurgatory)不断调用timer.add函数加入新的Task；另1方面，不是Timer内部有线程驱动，而是有一个外部线程ExpiredOperationReaper，不断调用timer.advanceClock函数，来驱动整个Timer。

同时，当某一个TimerTask到期之后，不是由Timer直接执行此TimerTask。而是交由一个executor，来执行所有过期的TimerTask。之所以这么做，是因为不能让TimerTask的执行阻塞Timer本身的进度。

总结一下：这里有2个外部线程，一个驱动Timer，一个executor，专门用来执行过期的Task。这2个线程，都是DelayedOperationPurgatory的内部变量。

```java
//TimingWheel内部结构
private[timer] class TimingWheel(tickMs: Long, wheelSize: Int, startMs: Long, taskCounter: AtomicInteger, queue: DelayQueue[TimerTaskList]) {

  private[this] val interval = tickMs * wheelSize   //每1格的单位 ＊ 总格数（比如1格是1秒，60格，那总共也就能表达60s)

  //核心变量之1：每个刻度对应一个TimerTask的链表
  private[this] val buckets = Array.tabulate[TimerTaskList](wheelSize) { _ => new TimerTaskList(taskCounter) }

  ...
  //核心变量之2：parent TimingWheel
  @volatile private[this] var overflowWheel: TimingWheel = null

//关键的TimingWheel的add函数
  def add(timerTaskEntry: TimerTaskEntry): Boolean = {
    val expiration = timerTaskEntry.timerTask.expirationMs

    if (timerTaskEntry.cancelled) {
      // 如果该任务已经被取消，则不加入timingWheel
      false
    } else if (expiration < currentTime + tickMs) {
      //如果该Task的过期时间已经小于当前时间 ＋ 基本的tick单位（1ms)，说明此任务已经过期了，不用再加入timingWheel
      false
    } else if (expiration < currentTime + interval) {
      // 如果过期时间 < 当前时间 ＋ interval，则说明当前的刻度盘可以表达此过期时间。这里的interval就是当前刻度盘所能表达的最大时间范围：tickMs * wheelSize

      //这里tickMs设置的是1ms，所以virtualId = expiration
      val virtualId = expiration / tickMs

      //关键的hash函数：根据过期时间，计算出bucket的位置
      val bucket = buckets((virtualId % wheelSize.toLong).toInt) 

      //把该Task加入bucket
      bucket.add(timerTaskEntry)

      //同一个bucket，所有task的expiration是相等的。因此，expiration相等的task，会hash到同1个bucket，然后此函数只第1次调用会成功
      if (bucket.setExpiration(virtualId * tickMs)) {
        queue.offer(bucket) //该桶只会被加入delayedQueue1次
      }
      true
    } else {
      //过期时间超出了currentTime + interval，说明该过期时间超出了当前刻度盘所能表达的最大范围，则调用其parent刻度盘，来试图加入此Task
      if (overflowWheel == null) addOverflowWheel()
      overflowWheel.add(timerTaskEntry)
    }
  }
```



**过期**

正如上面的图所示，外部线程每隔200ms调用1次advanceClock，从而驱动时钟不断运转。在驱动过程中，发现过期的Task，放入executors执行。

```java
  private class ExpiredOperationReaper extends ShutdownableThread(
    "ExpirationReaper-%d".format(brokerId),
    false) {

    override def doWork() {
      //不断循环，每200ms调用1次advanceClock
      timeoutTimer.advanceClock(200L)  
      ...
    }
}


  def advanceClock(timeoutMs: Long): Boolean = {
    //关键点：这里判断一个Task是否过期，其实还是用delayedQueue来判断的。而不是TimingWheel本事
    //过期的bucket会从队列的首部出对
    var bucket = delayQueue.poll(timeoutMs, TimeUnit.MILLISECONDS)
    if (bucket != null) {
      writeLock.lock()
      try {
        while (bucket != null) {
          //把timingWheel的进度，调整到队列首部的bucket的过期时间，也就是当前时间
          timingWheel.advanceClock(bucket.getExpiration())

          //清空bucket，执行bucket中每个Task的过期函数（执行方式就是把所有这些过期的Task，放入executors)
          bucket.flush(reinsert)

          //再次从队列首部拿下1个过期的bucket。如果没有，直接返回null。该函数不会阻塞
          bucket = delayQueue.poll()
        }
      } finally {
        writeLock.unlock()
      }
      true
    } else {
      false
    }
  }

//TimingWheel
  def advanceClock(timeMs: Long): Unit = {
    if (timeMs >= currentTime + tickMs) {
      //更新currentTime(把timeMs取整，赋给currentTime)
      currentTime = timeMs - (timeMs % tickMs)

      //更新parent timingWheel的currentTime
      if (overflowWheel != null) overflowWheel.advanceClock(currentTime)
    }
  }
```

**取消**

Task的取消，并不是在Timer里面实现的。而是TimerTask自身，定义了一个cancel函数。所谓cancel，就是自己把自己用TimerTaskEntryList这个双向链表中删除。

```java
trait TimerTask extends Runnable {

  val expirationMs: Long // timestamp in millisecond

  private[this] var timerTaskEntry: TimerTaskEntry = null

  def cancel(): Unit = {
    synchronized {
      if (timerTaskEntry != null) timerTaskEntry.remove()
      timerTaskEntry = null
    }
  }

  def remove(): Unit = {
    var currentList = list
    while (currentList != null) {
      currentList.remove(this)  //从链表中，把自己删掉
      currentList = list
    }
  }

 //remove函数。因为是双向链表，所以删除不需要遍历链表。删除复杂度是O(1)
  def remove(timerTaskEntry: TimerTaskEntry): Unit = {
    synchronized {
      timerTaskEntry.synchronized {
        if (timerTaskEntry.list eq this) {
          timerTaskEntry.next.prev = timerTaskEntry.prev
          timerTaskEntry.prev.next = timerTaskEntry.next
          timerTaskEntry.next = null
          timerTaskEntry.prev = null
          timerTaskEntry.list = null
          taskCounter.decrementAndGet()
        }
      }
    }
  }
```

**刻度盘的层次**

每个刻度盘都有个变量，记录currentTime。所有刻度盘的currentTime基本是相等的（会根据自己的tickMs取整）。advanceClock函数，就是更新这个currentTime。

在这里，不同的刻度盘单位其实都是ms。只是不同的刻度盘上，1格所代表的时间长度是不一样的。这里有个关系：

parent 刻度盘的1格表示的时间长度 ＝ child刻度盘的整个表盘所表示的时间范围

```java
  private[this] def addOverflowWheel(): Unit = {
    synchronized {
      if (overflowWheel == null) {
        overflowWheel = new TimingWheel(
          tickMs = interval,  //parent刻度盘的刻度tickMs = child刻度盘的整个表盘范围 interval(tickMs * wheelSize)
          wheelSize = wheelSize,
          startMs = currentTime,
          taskCounter = taskCounter,
          queue
        )
      }
    }
  }
```

因此，从底往上: tickMs = 1ms, wheelSize = 20格 

第1层刻度盘能表达的expiration的范围就是[currentTime, currentTime + tickMs*wheelSize]; //每1格1ms，总共20ms范围 

第2层刻度盘能表达的expiration的范围就是[currentTime, currentTime + tickMs*wheelSize*wheelSize]; //每1格20ms，总共400ms范围 

第3层刻度盘能表达的expiration的范围就是[currentTime, currentTime + tickMs*wheelSize*wheelSize*WheelSize]; //每1格400ms，总共8000ms范围

**DelayedQueue**

从上面代码中可以看到，添加/取消的时间复杂度都是O(1)。

并且在上面的代码中，大家可以看出，TimingWheel.advanceClock()函数里面其实什么都没做，就只是更新了一下所有刻度盘的currentTime。真正的判断哪个Task过期的逻辑，其实是用DelayedQueue来判断的，而不是通过TimingWheel判断的。

那TimingWheel在此处到底起了一个什么作用呢？

让我们从另外1个角度，来画一下Timer的实现机制：

前面讲过，expiration相等的TimerTask，会组成一个双向链表，称之为一个bucket。DelayedQueue的每个节点，放入的就是一个bucket，而不是单个的TimerTask。过期的判断，就是通过DelayedQueue来实现的。


## Log文件结构

**每个topic_partition对应一个目录**

假设有一个topic叫my_topic，3个partition，分别为my_topic_0, my_topic_1, my_topic_2。其中前2个parition存在1台机器A，另外一个paritition存在另1台机器B上（这里A, B都是指对应partition的leader机器）。

则机器A上，就有2个对应的目录my_topic_0, my_topic_1，也就是每个目录对应一个topic_partition，目录名字就是topic_partition名字。

而目录中，日志文件按照大小或者时间回滚，文件名字为00000000.kafka, 1200000.kakfa。。。

文件名字就是该文件中第1条消息对应的文件offset地址，也即是说，该message之前的所有消息，共占用了1200000字节。因为从逻辑上来讲，你可以认为整个partition就对应1个大的log文件，只是实现层面，回滚成了多个文件。

同时，Kafka中有一个配置参数

log.dir   //缺省为/tmp/kafka-logs

**文件offset作为Message ID**

正如上面所说，kafka没有额外用类似uuid的方式，为每条消息生成一个唯一的message id，而是直接用该message在文件中的offset，作为该message的id。

这样做有一个很大的好处，根据message的offset，也就是id，很容易定位到message在文件中的位置。

具体说来，就是：首先可以2分查找，定位出在哪1个文件里面；然后减去文件名字对应的offset数值，就得到该message在该文件中对应的offset。

**变长消息存储**

我们知道，kafka的消息是变长的。对于变长记录的存储，一般都是在记录的最前面，用固定的几个字节（比如4个字节），来存储记录的长度。

读的时候，先读取这固定的4个字节，获得记录长度，再根据长度读取后面内容。

**flush刷盘机制**

熟悉Linux操作系统原理的都知道，当我们把数据写入到文件系统之后，数据其实在操作系统的page cache里面，并没有刷到磁盘上去。如果此时操作系统挂了，其实数据就丢了。

一方面，应用程序可以调用fsync这个系统调用来强制刷盘；另一方面，操作系统有后台线程，定期刷盘。

如果应用程序每写入1次数据，都调用一次fsync，那性能损耗就很大，所以一般都会在性能和可靠性之间进行权衡。因为对应一个应用来说，虽然应用挂了，只要操作系统不挂，数据就不会丢。

另外, kafka是多副本的，当你配置了同步复制之后。多个副本的数据都在page cache里面，出现多个副本同时挂掉的概率比1个副本挂掉，概率就小很多了。

对于kafka来说，也提供了相关的配置参数，来让你在性能与可靠性之间权衡：

## Controller state machine and new Controller design

**PartitionStateChange:** 

Valid states are:

1. NonExistentPartition: This state indicates that the partition was either never created or was created and then deleted.
2. NewPartition: After creation, the partition is in the NewPartition state. In this state, the partition should have replicas assigned to it, but no leader/isr yet.
3. OnlinePartition: Once a leader is elected for a partition, it is in the OnlinePartition state.
4. OfflinePartition: If, after successful leader election, the leader for partition dies, then the partition moves to the OfflinePartition state.

Valid state transitions are:

NonExistentPartition -> NewPartition
1. load assigned replicas from ZK to controller cache

NewPartition -> OnlinePartition
1. assign first live replica as the leader and all live replicas as the isr; write leader and isr to ZK
2. for this partition, send **LeaderAndIsr** request to every live replica and UpdateMetadata request to every live broker

OnlinePartition,OfflinePartition -> OnlinePartition
1. select new leader and isr for this partition and a set of replicas to receive the LeaderAndIsr request, and write leader and isr to ZK

- OfflinePartitionLeaderSelector: new leader = a live replica (preferably in isr); new isr = live isr if not empty or just the new leader if otherwise; receiving replicas = live assigned replicas
- ReassignedPartitionLeaderSelector: new leader = a live reassigned replica; new isr = current isr; receiving replicas = reassigned replicas
- PreferredReplicaPartitionLeaderSelector: new leader = first assigned replica (if in isr); new isr = current isr; receiving replicas = assigned replicas
- ControlledShutdownLeaderSelector: new leader = replica in isr that's not being shutdown; new isr = current isr - shutdown replica; receiving replicas = live assigned replicas
2. for this partition, send LeaderAndIsr request to every receiving replica and UpdateMetadata request to every live broker

NewPartition,OnlinePartition -> OfflinePartition
1. nothing other than marking partition state as Offline

OfflinePartition -> NonExistentPartition
1. nothing other than marking the partition state as NonExistentPartition

**ReplicaStateChange:**

Valid states are:

1. NewReplica: When replicas are created during topic creation or partition reassignment. In this state, a replica can only get become follower state change request. 
2. OnlineReplica: Once a replica is started and part of the assigned replicas for its partition, it is in this state. In this state, it can get either become leader or become follower state change requests.
3. OfflineReplica : If a replica dies, it moves to this state. This happens when the broker hosting the replica is down.
4. NonExistentReplica: If a replica is deleted, it is moved to this state.

Valid state transitions are:

NonExistentReplica --> NewReplica
1. send LeaderAndIsr request with current leader and isr to the new replica and UpdateMetadata request for the partition to every live broker

NewReplica-> OnlineReplica
1. add the new replica to the assigned replica list if needed

OnlineReplica,OfflineReplica -> OnlineReplica
1. send LeaderAndIsr request with current leader and isr to the new replica and UpdateMetadata request for the partition to every live broker

NewReplica,OnlineReplica -> OfflineReplica
1. send StopReplicaRequest to the replica (w/o deletion)
2. remove this replica from the isr and send LeaderAndIsr request (with new isr) to the leader replica and UpdateMetadata request for the partition to every live broker.

OfflineReplica -> NonExistentReplica
1. send StopReplicaRequest to the replica (with deletion)

**KafkaController Operations:**

onNewTopicCreation:
1. call onNewPartitionCreation

onNewPartitionCreation:
1. new partitions -> NewPartition
2. all replicas of new partitions -> NewReplica
3. new partitions -> OnlinePartition
4. all replicas of new partitions -> OnlineReplica

onBrokerFailure:
1. partitions w/o leader -> OfflinePartition
2. partitions in OfflinePartition and NewPartition -> OnlinePartition (with OfflinePartitionLeaderSelector)
3. each replica on the failed broker -> OfflineReplica

onBrokerStartup:
1. send UpdateMetadata requests for all partitions to newly started brokers
2. replicas on the newly started broker -> OnlineReplica
3. partitions in OfflinePartition and NewPartition -> OnlinePartition (with OfflinePartitionLeaderSelector)
4. for partitions with replicas on newly started brokers, call onPartitionReassignment to complete any outstanding partition reassignment

onPartitionReassignment: (OAR: old assigned replicas; RAR: new re-assigned replicas when reassignment completes)

1. update assigned replica list with OAR + RAR replicas
2. send **LeaderAndIsr** request to every replica in OAR + RAR (with AR as OAR + RAR)
3. replicas in RAR - OAR -> NewReplica
4. wait until replicas in RAR join isr
5. replicas in RAR -> OnlineReplica
6. set AR to RAR in memory
7. send LeaderAndIsr request with a potential new leader (if current leader not in RAR) and a new assigned replica list (using RAR) and same isr to every broker in RAR
8. replicas in OAR - RAR -> Offline (force those replicas out of isr)
9. replicas in OAR - RAR -> NonExistentReplica (force those replicas to be deleted)
10. update assigned replica list to RAR in ZK
11. update the /admin/reassign_partitions path in ZK to remove this partition
12. after electing leader, the replicas and isr information changes, so resend the update metadata request to every broker

For example, if OAR = {1, 2, 3} and RAR = {4,5,6}, the values in the assigned replica (AR) and leader/isr path in ZK may go through the following transition.

**State change input source:**

Listeners Registered to Zookeeper.

1. AddPartitionsListener
2. BrokerChangeListener
3. DeleteTopicListener
4. PartitionReassignedListener(Admin)
5. PreferredReplicaElectionListener(Admin)
6. ReassignedPartitionsIsrChangeListener
7. TopicChangeListener

Channels to brokers (controlled shutdown)

Internal scheduled tasks (preferred leader election)

**Problems of existing controller**

1. State change are executed by different listeners concurrently. Hence complicated synchronization is needed which is error prone and difficult to debug.
2. State propagation is not synchronized. Brokers might be in different state for undetermined time. This leads to unnecessary extra data loss.
3. During controlled shutdown process, two connections are used for controller to broker communication. This makes state change propagation and controlled shutdown approval out of order.
4. Some of the state changes are complicated because the **ReplicaStateMachine** and **PartitionStateMachine** are separate but the state changes themselves are not. So the controller has to maintain some sort of ordering between the state changes executed by the individual state machines. **In the new design, we will look into folding them into a single one.**
5. Many state changes are executed for one partition after another. It would be much more efficient to do it in one request.
6. Some long running operations need to support cancellation. For example, we want to support cancellation of partition reassignment for those partitions whose reassignment hasn't started yet. The new controller should be able to abort/cancel the long running process.

**New controller design**

We are going to change the state change propagation model, state change execution and output of the state change input source. More specifically:
1. Abstract the output of each state change input source to an event.
2. Have a single execution thread to serially process events one at a time.
3. The zk listeners are responsible for only context updating but not event execution.
4. Use o.a.k.clients.NetworClient + callback for state change propagation.

## Partition 的存储

上面提高了 paritition 的存储，但是不够详细，并且关键点都没给出

segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件.

segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

以上述图2中一对segment file文件为例，说明segment中index<—->data file对应关系物理结构如下：

![](/images/posts/kafka/kafka-fs-segment-file-list-small.png)

![](/images/posts/kafka/kafka-fs-index-correspond-data.png)

上述图3中索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。
其中以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)、以及该消息的物理偏移地址为497。

从图 3 可以看出，索引其实是有两层的，第一层是 index 和 log, 第二层是 .index 内的内容。

**在partition中如何通过offset查找message**

例如读取offset=368776的message，需要通过下面2个步骤查找。

1. 第一步查找segment file
上述图2为例，其中00000000000000000000.index表示最开始的文件，起始偏移量(offset)为0.第二个文件00000000000000368769.index的消息量起始偏移量为368770 = 368769 + 1.同样，第三个文件00000000000000737337.index的起始偏移量为737338=737337 + 1，其他后续文件依次类推，以起始偏移量命名并排序这些文件，只要根据offset **二分查找**文件列表，就可以快速定位到具体文件。
当offset=368776时定位到00000000000000368769.index|log

2. 第二步通过segment file查找message
通过第一步定位到segment file，当offset=368776时，依次定位到00000000000000368769.index的元数据物理位置和00000000000000368769.log的物理偏移地址，然后再通过00000000000000368769.log顺序查找直到offset=368776为止。

从上述图3可知这样做的优点，segment index file采取稀疏索引存储方式，它减少索引文件大小，通过mmap可以直接内存操作，稀疏索引为数据文件的每个对应message设置一个元数据指针,它比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间。

**磁盘读写**

从上述图5可以看出，Kafka运行时很少有大量读磁盘的操作，主要是定期批量写磁盘操作，因此操作磁盘很高效。这跟Kafka文件存储中读写message的设计是息息相关的。Kafka中读写message有如下特点:

写message

1. 消息从java堆转入page cache(即物理内存)。
2. 由异步线程刷盘,消息从page cache刷入磁盘。

读message

1. 消息直接从page cache转入socket发送出去。
2. 当从page cache没有找到相应数据时，此时会产生磁盘IO,从磁
3. 盘Load消息到page cache,然后直接从socket发出去

Kafka高效文件存储设计特点

1. Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用
2. 通过索引信息可以快速定位message和确定response的最大大小
3. 通过index元数据全部**映射到memory**，可以避免 segment file 的 IO 磁盘操作
4. 通过索引文件稀疏存储，可以大幅降低 index 文件元数据占用空间大小

