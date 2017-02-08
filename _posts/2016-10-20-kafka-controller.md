---
layout: post
title:  "kafka controller"
date:   "2016-10-20 00:00:00"
categories: kafka
keywords: kafka
---

### Partition

Partition 包含了很多的元素, 包括比较重要的 ISR, AR, replicaManager 和 LogManager 等等

### Update ISR 和 offset

KafkaApis 收到 FetchRequest 以后

```scala
def handle(request: RequestChannel.Request) {
    case RequestKeys.FetchKey => handleFetchRequest(request)

def handleFetchRequest(request: RequestChannel.Request) {
    val fetchRequest = request.requestObj.asInstanceOf[FetchRequest]
      replicaManager.fetchMessages(
        fetchRequest.maxWait.toLong,
        fetchRequest.replicaId,
        fetchRequest.minBytes,
        authorizedRequestInfo,
        sendResponseCallback)
case class FetchRequest(versionId: Short = FetchRequest.CurrentVersion,
                        correlationId: Int = FetchRequest.DefaultCorrelationId,
                        clientId: String = ConsumerConfig.DefaultClientId,
                        replicaId: Int = Request.OrdinaryConsumerId,
                        maxWait: Int = FetchRequest.DefaultMaxWait,
                        minBytes: Int = FetchRequest.DefaultMinBytes,
                        requestInfo: Map[TopicAndPartition, PartitionFetchInfo])
        extends RequestOrResponse(Some(RequestKeys.FetchKey))

case class PartitionFetchInfo(offset: Long, fetchSize: Int)                
```

在 fetchMessages 中, 会进行 OffsetUpdate 和 ISR update 操作
返回给 client 的信息叫做 FetchResponsePartitionData, 包含 hw, messageSet

```scala
def fetchMessages
    val logReadResults = readFromLocalLog(fetchOnlyFromLeader, fetchOnlyCommitted, fetchInfo)
    if(Request.isValidBrokerId(replicaId))
      updateFollowerLogReadResults(replicaId, logReadResults)
      
    if is delayed fetch

      // construct the fetch results from the read results
      val fetchPartitionStatus = logReadResults.map { case (topicAndPartition, result) =>
        (topicAndPartition, FetchPartitionStatus(result.info.fetchOffsetMetadata, fetchInfo.get(topicAndPartition).get))
      }

      val fetchMetadata = FetchMetadata(fetchMinBytes, fetchOnlyFromLeader, fetchOnlyCommitted, isFromFollower, fetchPartitionStatus)

      val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, responseCallback)

      // create a list of (topic, partition) pairs to use as keys for this delayed fetch operation
      val delayedFetchKeys = fetchPartitionStatus.keys.map(new TopicPartitionOperationKey(_)).toSeq

      // try to complete the request immediately, otherwise put it into the purgatory;
      // this is because while the delayed fetch operation is being created, new requests
      // may arrive and hence make this operation completable.
      // 一个 delayedFetch 可能会有很多的 Key 对应到它
      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
```

先看 offset 和 isr 的管理, 再看 purgatory 对 delayedFetch 的管理

```scala
private def updateFollowerLogReadResults(replicaId: Int, readResults: Map[TopicAndPartition, LogReadResult])
    partition.updateReplicaLogReadResult(replicaId, readResult)
    tryCompleteDelayedProduce(new TopicPartitionOperationKey(topicAndPartition))

def updateReplicaLogReadResult(replicaId: Int, logReadResult: LogReadResult)
    replica.updateLogReadResult(logReadResult)
    maybeExpandIsr(replicaId)

// 更新某一个 replica 的 offset
def updateLogReadResult(logReadResult : LogReadResult) {
    logEndOffset = logReadResult.info.fetchOffsetMetadata

    /* If the request read up to the log end offset snapshot when the read was initiated,
     * set the lastCaughtUpTimeMsUnderlying to the current time.
     * This means that the replica is fully caught up.
     */
    if(logReadResult.isReadFromLogEnd) {
      lastCaughtUpTimeMsUnderlying.set(time.milliseconds)
    }
  }

def maybeExpandIsr(replicaId: Int)
    val replica = getReplica(replicaId).get
    val leaderHW = leaderReplica.highWatermark
    assignedReplicas.map(_.brokerId).contains(replicaId) &&
        replica.logEndOffset.offsetDiff(leaderHW) >= 0) {

        val newInSyncReplicas = inSyncReplicas + replica

         // update ISR in ZK and cache
         updateIsr(newInSyncReplicas)
         replicaManager.isrExpandRate.mark()
         maybeIncrementLeaderHW(leaderReplica)

private def maybeIncrementLeaderHW(leaderReplica: Replica): Boolean = 
    val allLogEndOffsets = inSyncReplicas.map(_.logEndOffset)
    val newHighWatermark = allLogEndOffsets.min(new LogOffsetMetadata.OffsetOrdering)
    val oldHighWatermark = leaderReplica.highWatermark
    if(oldHighWatermark.precedes(newHighWatermark))
        eaderReplica.highWatermark = newHighWatermark
```

follower 或者 consumer 读取 localLog 以后, replica 上的 logEndOffset 得到更新, 然后 Leader 去查找所有 replica 的 
logEndOffset 信息, 找到以后更新 leader 的 ISR

这里解释了两点, 第一点是 non-leader replica 本身只会记录自己的 hw, 而不会记录 LogEndOffset, 因为对他来说, 自己能够记录下来的
信息, 肯定是 leader 都记录好的了。但是 leader 要记录每个 replica 的 logEndOffset 以便了解整个集群的状态
第二点是, 每个 replica 从 leader 获取数据的时候, 会把自己想要的 offset 发送出去, 从 localLog 读取成功后, offset 增加, 这个时候,
leader 会根据所有节点的 logEndOffset 更新 ISR, 更新 HW

### DelayedFetch 的处理

紧跟着上面的代码, 上面的代码有多处和 delayed fetch 有关, 第一处是 updateFollowLogReadResults 时, 会调用 
tryCompleteDelayedProduce 方法, 这是因为, Producer 发送的消息, 在 ack=-1 的情况下, producer 要等到所有的
replica 都复制完毕

```scala
  def tryCompleteDelayedProduce(key: DelayedOperationKey) {
    val completed = delayedProducePurgatory.checkAndComplete(key)
    debug("Request key %s unblocked %d producer requests.".format(key.keyLabel, completed))
  }

  def checkAndComplete(key: Any): Int = {
    val watchers = inReadLock(removeWatchersLock) { watchersForKey.get(key) }
    if(watchers == null)
      0
    else
      watchers.tryCompleteWatched()
  }
```

tryCompleted 会调用 operations 的 tryComplete, DelayedProduce 的 tryComplete 代码如下

```scala
  /**
   * The delayed produce operation can be completed if every partition
   * it produces to is satisfied by one of the following:
   *
   * Case A: This broker is no longer the leader: set an error in response
   * Case B: This broker is the leader:
   *   B.1 - If there was a local error thrown while checking if at least requiredAcks
   *         replicas have caught up to this operation: set an error in response
   *   B.2 - Otherwise, set the response with no error.
   */
  override def tryComplete(): Boolean = {
    // check for each partition if it still has pending acks
    produceMetadata.produceStatus.foreach { case (topicAndPartition, status) =>
      trace("Checking produce satisfaction for %s, current status %s"
        .format(topicAndPartition, status))
      // skip those partitions that have already been satisfied
      if (status.acksPending) {
        val partitionOpt = replicaManager.getPartition(topicAndPartition.topic, topicAndPartition.partition)
        val (hasEnough, errorCode) = partitionOpt match {
          case Some(partition) =>
            partition.checkEnoughReplicasReachOffset(status.requiredOffset)
          case None =>
            // Case A
            (false, ErrorMapping.UnknownTopicOrPartitionCode)
        }
        if (errorCode != ErrorMapping.NoError) {
          // Case B.1
          status.acksPending = false
          status.responseStatus.error = errorCode
        } else if (hasEnough) {
          // Case B.2
          status.acksPending = false
          status.responseStatus.error = ErrorMapping.NoError
        }
      }
    }

    // check if each partition has satisfied at lease one of case A and case B
    if (! produceMetadata.produceStatus.values.exists(p => p.acksPending)) forceComplete()
    else false
  }
```

正常情况下, 当 replia 全部收集到了数据即可返回 response 给 producer

```scala
def checkEnoughReplicasReachOffset(requiredOffset: Long): (Boolean, Short) = {
    val curInSyncReplicas = inSyncReplicas
    val numAcks = curInSyncReplicas.count(r => {
        if (r.logEndOffset.messageOffset >= requiredOffset) { true
        else false
    val minIsr = leaderReplica.log.get.config.minInSyncReplicas
    if (leaderReplica.highWatermark.messageOffset >= requiredOffset ) 
```

判断逻辑有些迷糊, 到底是用 ISR的 endOffset 还是 leader 的 HW
 
### LeaderAndISR change 

从整个集群的所有Brokers中选举出一个Controller，它主要负责：

- Partition 的 Leader变化事件
- 新创建或删除一个topic
- 重新分配Partition
- 管理分区的状态机和副本的状态机

当控制器完成决策之后（决定了Partition新的Leader和ISR），它会将这个决策持久化到ZK中（将LeaderAndISR写入到ZK节点），
并且向所有受到影响的Brokers通过直接RPC的形式直接发送新的决策（发送LeaderAndISRRequest）。

每个KafkaServer中都会创建一个KafkaController对象，但是集群中只允许存在一个Leader Controller，
这是通过ZooKeeper选举出来的：第一个成功创建zk节点的那个Controller会成为Leader，其他的Controller会一直存在，
但是并不会发挥作用。只有当原先的Controller挂掉后，才会选举出新的Controller。集群中所有Brokers选举一个Controller
和Partition中所有Replicas选举一个Leader是类似的。

Broker失败后Controller端的处理步骤如下：

- 从ZK中读取现存的brokers
- broker宕机，会引起partition的Leader或ISR变化，获取在在这个broker上的partitions：SET_PARTITION
- 对set_p的每个Partition P

    3.1 从ZK的leaderAndISR节点读取P的当前ISR
    
    3.2 决定P的新Leader和新ISR（优先级分别是ISR中存活的broker，AR中任意存活的作为Leader）
    
    3.3 将P最新的leader，ISR，epoch回写到ZK的leaderAndISR节点
    
- 将SET_PARTITION中每个Partition的LeaderAndISR指令（包括最新的leaderAndISR数据）发送给受到影响的brokers

```scala
case class LeaderAndIsr(var leader: Int, var leaderEpoch: Int, var isr: List[Int], var zkVersion: Int)

def handleLeaderAndIsrRequest(request: RequestChannel.Request) {
    val leaderAndIsrRequest = request.requestObj.asInstanceOf[LeaderAndIsrRequest]
    // for each new leader or follower, call coordinator to handle consumer group migration.
    val result = replicaManager.becomeLeaderOrFollower(leaderAndIsrRequest, metadataCache, onLeadershipChange)
```

coordinator 负责处理 cluster of consumer 的关系

```scala
def becomeLeaderOrFollower(leaderAndISRRequest: LeaderAndIsrRequest,
                             metadataCache: MetadataCache,
                             onLeadershipChange: (Iterable[Partition], Iterable[Partition]) => Unit): BecomeLeaderOrFollowerResult = {
    val partitionsBecomeLeader = if (!partitionsTobeLeader.isEmpty)
              makeLeaders(controllerId, controllerEpoch, partitionsTobeLeader, leaderAndISRRequest.correlationId, responseMap)
    val partitionsBecomeFollower = if (!partitionsToBeFollower.isEmpty)
              makeFollowers(controllerId, controllerEpoch, partitionsToBeFollower, leaderAndISRRequest.correlationId, responseMap, metadataCache)
    if (!hwThreadInitialized) {
        startHighWaterMarksCheckPointThread()
        hwThreadInitialized = true
    replicaFetcherManager.shutdownIdleFetcherThreads()
    onLeadershipChange(partitionsBecomeLeader, partitionsBecomeFollower)
    BecomeLeaderOrFollowerResult(responseMap, ErrorMapping.NoError)
```

Make the current broker to become leader for a given set of partitions by:

1. Stop fetchers for these partitions
2. Update the partition metadata in cache
3. Add these partitions to the leader partitions set

Make the current broker to become follower for a given set of partitions by:

1. Remove these partitions from the leader partitions set.
2. Mark the replicas as followers so that no more data can be added from the producer clients.
3. Stop fetchers for these partitions so that no more data can be added by the replica fetcher threads.
4. Truncate the log and checkpoint offsets for these partitions.
5. If the broker is not shutting down, add the fetcher to the new leaders.

总结下来, 无非是开关 partitionFetcher, 修改 partitionFetcher 指向的位置, 修改 MetaData in cache, 

### 数据一致性保证

一致性定义: 若某条消息对Consumer可见,那么即使Leader宕机了,在新Leader上数据依然可以被读到

1. HighWaterMark简称HW: Partition的高水位，取一个partition对应的ISR中最小的LEO作为HW，
消费者最多只能消费到HW所在的位置，另外每个replica都有highWatermark，leader和follower各自负责更
新自己的highWatermark状态，highWatermark <= leader. LogEndOffset

- 2.对于Leader新写入的msg，Consumer不能立刻消费，Leader会等待该消息被所有ISR中的replica同步后, 
更新HW,此时该消息才能被Consumer消费，即Consumer最多只能消费到HW位置

### Partition Recovery机制

每个Partition会在磁盘记录一个 RecoveryPoint, 记录已经flush到磁盘的最大offset。当broker fail 重启时,
会进行loadLogs。 首先会读取该Partition的RecoveryPoint,找到包含RecoveryPoint的segment及以后的segment, 
这些segment就是可能没有 完全flush到磁盘segments。然后调用segment的recover,重新读取各个segment的msg,并重建索引

1. 以segment为单位管理Partition数据,方便数据生命周期的管理,删除过期数据简单
2. 在程序崩溃重启时,加快recovery速度,只需恢复未完全flush到磁盘的segment
3. 通过index中offset与物理偏移映射,用二分查找能快速定位msg,并且通过分多个Segment,每个index文件很小,查找速度更快。

### leadership 改变时，offset 超过 HW

通常情况下，follower的HW总是落后于leader的HW。所以在leader变化时，消费者发送的offset中可能会落在
新leader的HW和LEO之间（因为消费者的消费进度依赖于Leader的HW，旧Leader的HW比较高，而原先的follower
作为新Leader，它的HW还是落后于旧Leader，所以消费者的offset虽然比旧Leader的HW低，但是有可能
比新Leader的HW要大）。如果没有发生leader变化，服务端会返回OffsetOutOfRangeException给客户端。但这
种情况下如果请求的offset介于HW和LEO之间，服务端会返回空消息集给消费者。

注册在ZooKeeper上的监听器

1. BrokerChangeListener（Broker挂掉）
2. DeleteTopicListener（删除Topic）
3. TopicChangeListener（更改Topic）
4. AddPartitionsListener（为Topic新增加Partition）
5. PartitionReassignedListener（Admin）
6. PreferredReplicaElectionListener（Admin）
7. ReassignedPartitionsIsrChangeListener

注意有可能新 leader的 HW 会比之前的leader的HW要落后，这是因为新leader有可能是ISR，也有可能是AR中的replica。
而原先作为 Follower 的 replica，它的 HW 只会在向 Leader 发送抓取请求时, Leader 在抓取响应中除了返回消息也
会附带自己的HW给follower，Follower收到消息和HW后，才会更新自己的replica的HW，这中间有一定的时间间隔会
导致Follower的HW会比Leader的HW要低。因此Follower在转变为Leader之后，它的HW是有可能比老的Leader的HW要低的。
如果在leader角色转变之后，一个消费者客户端请求的offset可能比新的Leader的HW要大（因为消费者最多只
消费到Leader的HW位置，但是消费者并不关心Leader到底有没有变化，所以如果旧的Leader的HW=10，那么客
户端就可以消费到offset=10这个位置，而Leader发生转变后，HW可能降低为9，而这个时候客户端继续发
送offset=10，就有可能比Leader的HW要大了！）。这种情况下，如果消费者要消费Leader的HW到LEO之间
的数据，Broker会返回空的集合，而如果消费者请求的offset比LEO还要大，就会抛出OffsetOutofRangeException（LEO表示
的是日志的最新位置，HW比LEO要小，客户端只能消费到HW位置，更不可能消费到LEO了)

对于要转变为Follower的replica，原先如果是Leader的话，则要停止提交线程，由于当前Replica的leader可能会发生变化，
所以在开始时要停止抓取线程，在最后要新创建到Replica最新leader的抓取线程，这中间还要截断日志到Replica的HW位置。

### broker 故障客户端如何路由

所以可能你会认为不要让  controller 先更新 leaderAndISR 节点，而是发送指令给 brokers，每个 broker 在
收到指令并处理完成后才，让每个broker来更新这个节点的数据。不过这里让controller来更新leaderAndISR节点
是有原因的：我们是依赖ZK的leaderAndISR节点来保持controller和partition的leader的同步。当controller选
举出新的leader之后，它并不希望ISR（新leader的ISR）被当前的leader（旧的leader）改变。否则的话（假设
ISR可以被旧leader改变），新选举出来的leader在接管正式成为leader之前可能会被当前leader从ISR中剔除
出去。通过立即将新leader发布到leaderAndISR节点，controller能够防止当前leader更新ISR（选举
新的leader在写到ZK时，epoch增加，而如果当前leader想要更新ISR比如将新选举的leader从ISR中剔除掉，因
为epoch不匹配，所以当前leader就不再有机会更新ISR了)

### ZKQueue vs 直接RPC

通过ZK完成Controller和brokers的通信是非常低效的，因为每次通信都需要2次ZK写（每个写
操作花费2个RPC），一次监听器的触发（写之后数据改变触发监听器调用），一次ZK读（监听器
要获取最新的写操作结果），所以对于一次通信就花费了6次RPC（2*2+1+1=6，RPC指的是在不同节点之
间的通信，controller和brokers是在不同节点上，需要通过RPC通信才能读写数据）。

如果controler和所有的brokers之间为了能够直接通信，在Broker端实现一个admin RPC，那么每次通
信只需要一个RPC。使用RPC意味着当一个broker当掉后，它可能会丢失一些来自Controller的命令，所
以在broker启动时它需要读取zk中保存的状态信息来恢复自己的状态。

### 副本放置策略

kafka 将一个逻辑意义的 topic 分成多个物理意义上的 partition，每个 partition 是这个 topic 的一部分消息，
将 partition分布在多个brokers上，可以达到负载均衡的目的。除了Partition的分配，每个Partition都有Replcias，
所以也要考虑Replica的分配策略，因为不能把一个Partition的所有Replicas都放在同一台机器上。一般我们希望将
所有的Partition能够均匀地分布在集群中所有的Brokers上。Kafka分配Replica的算法如下：

- 将所有存活的N个Brokers和待分配的Partition排序
- 将第i个Partition分配到第(i mod n)个Broker上，这个Partition的第一个Replica存在于这个分配的Broker上，并且会作为partition的优先副本
- 将第i个Partition的第j个Replica分配到第((i + j) mod n)个Broker上

### Follower failure

如果Follower Replica失败超过一定时间后，Leader会将这个失败的follower从ISR中移除（follower没有发送fetch请求）。
由于ISR保存的是所有全部赶得上Leader的follower replicas，失败的follower肯定是赶不上了。虽然ISR现
在少了一个，但是并不会引起的数据的丢失，ISR中剩余的replicas会继续同步数据（只要ISR中有一个follower，就
不会丢失数据，实际上leader replica也是在ISR中的，所以即使所有的follower都挂掉了，只要leader没有问题，也
不会出现问题的）。

当失败的follower恢复时，它首先将自己的日志截断到上次checkpointed时刻的HW (checkpoint是
每个broker节点都有的）。因为checkpoint记录的是所有Partition的hw offset。当follower失败时,
checkpoint中关于这个Partition的HW就不会再更新了。而这个时候存储的 HW 信息和 follower partition 
replica的offset并不一定是一致的。比如这个follower获取消息比较快，但是ISR中有其他follower复制消息
比较慢，这样Leader并不会很快地更新HW，这个快的follower的hw也不会更新 (leader广播hw给follower)
这种情况下，这个follower日志的offset是比hw要大的。所以在它恢复之后，要将比hw多的部分截掉，
然后继续从leader拉取消息（跟平时一样）。实际上，ISR中的每个follower日志的offset一定是比hw大的.
因为只有ISR中所有follower都复制完消息，leader才会增加hw，而每个Replica复制消息后，都会增加自己的offset.
也就是说有可能有些follower复制完了，而有另外一些follower还没有复制完，那么hw是不会增加的.

### add partition

Be aware that one use case for partitions is to semantically partition data, 
and adding partitions doesn't change the partitioning of existing data so this may 
disturb consumers if they rely on that partition. That is if data is partitioned 
by hash(key) % number_of_partitions then this partitioning will potentially be shuffled 
by adding partitions but Kafka will not attempt to automatically redistribute data in any way.


