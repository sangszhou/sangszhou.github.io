---
layout: post
title: Kafka consumer
categories: [kafka]
keywords: kafka consumer
---


针对版本  0.9 以上

## 使用例子 

```java
public class ConsumerLoop implements Runnable {
  private final KafkaConsumer<String, String> consumer;
  private final List<String> topics;
  private final int id;

  public ConsumerLoop(int id, String groupId, List<String> topics) {
    this.id = id;
    this.topics = topics;
    Properties props = new Properties();
    props.put("bootstrap.servers", "localhost:9092");
    props.put("group.id", groupId);
    props.put("key.deserializer", StringDeserializer.class.getName());
    props.put("value.deserializer", StringDeserializer.class.getName());
    this.consumer = new KafkaConsumer<>(props);
  }
 
  @Override
  public void run() {
    try {
      consumer.subscribe(topics);

      while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
        
        for (ConsumerRecord<String, String> record : records) {
          Map<String, Object> data = new HashMap<>();
          data.put("partition", record.partition());
          data.put("offset", record.offset());
          data.put("value", record.value());
          System.out.println(this.id + ": " + data);
        }
      }
    } catch (WakeupException e) {
      // ignore for shutdown 
    } finally {
      consumer.close();
    }
  }

  public void shutdown() {
    consumer.wakeup();
  }
}
```

### using command line tool

```
# bin/kafka-topics.sh --create --topic consumer-tutorial --replication-factor 1 --partitions 3 --zookeeper localhost:2181

# bin/kafka-verifiable-producer.sh --topic consumer-tutorial --max-messages 200000 --broker-list localhost:9092
```

默认情况下，topic 会自动创建，但是第一次发送到为创建 topic 的消息会丢失。

```java
public static void main(String[] args) { 
  int numConsumers = 3;
  String groupId = "consumer-tutorial-group"
  List<String> topics = Arrays.asList("consumer-tutorial");
  ExecutorService executor = Executors.newFixedThreadPool(numConsumers);

  final List<ConsumerLoop> consumers = new ArrayList<>();
  for (int i = 0; i < numConsumers; i++) {
    ConsumerLoop consumer = new ConsumerLoop(i, groupId, topics);
    consumers.add(consumer);
    executor.submit(consumer);
  }

  Runtime.getRuntime().addShutdownHook(new Thread() {
    @Override
    public void run() {
      for (ConsumerLoop consumer : consumers) {
        consumer.shutdown();
      } 
      
      executor.shutdown();
      
      try {
        executor.awaitTermination(5000, TimeUnit.MILLISECONDS);
      } catch (InterruptedException e) {
        e.printStackTrace;
      }
    }
  });
}
```

### Consumer Liveness

When part of a consumer group, each consumer is assigned a subset of the partitions from topics it has subscribed to. This is basically a group lock on those partitions. As long as the lock is held, no other members in the group will be able to read from them. When your consumer is healthy, this is exactly what you want. It’s the only way that you can avoid duplicate consumption. But if the consumer dies due to a machine or application failure, you need that lock to be released so that the partitions can be assigned to a healthy member. 

Kafka’s group coordination protocol addresses this problem using a heartbeat mechanism. After every rebalance, all members of the current generation begin sending periodic heartbeats to the group coordinator. As long as the coordinator continues receiving heartbeats, it assumes that members are healthy. On every received heartbeat, the coordinator starts (or resets) a timer. If no heartbeat is received when the timer expires, the coordinator marks the member dead and signals the rest of the group that they should rejoin so that partitions can be reassigned. The duration of the timer is known as the session timeout and is configured on the client with the setting session.timeout.ms. 

The session timeout ensures that the lock will be released if the machine or application crashes or if a network partition isolates the consumer from the coordinator. However, application failures are a little trickier to handle generally. Just because the consumer is still sending heartbeats to the coordinator does not necessarily mean that the application is healthy. 

The consumer’s poll loop is designed to handle this problem. All network IO is done in the foreground when you call poll or one of the other blocking APIs. The consumer does not use any background threads. This means that heartbeats are only sent to the coordinator when you call **poll**. If your application stops polling (whether because the processing code has thrown an exception or a downstream system has crashed), then no heartbeats will be sent, the session timeout will expire, and the group will be rebalanced. 

The only problem with this is that a spurious rebalance might be triggered if the consumer takes longer than the session timeout to process messages. You should therefore set the session timeout large enough to make this unlikely. The default is 30 seconds, but it’s not unreasonable to set it as high as several minutes. The only downside of a larger session timeout is that it will take longer for the coordinator to detect genuine consumer crashes.

### Delivery Semantics

When a consumer group is first created, the initial offset is set according to the policy defined by the auto.offset.reset configuration setting. Once the consumer begins processing, it commits offsets regularly according to the needs of the application. After every subsequent rebalance, the position will be set to the last committed offset for that partition in the group. If the consumer crashes before committing offsets for messages that have been successfully processed, then another consumer will end up **repeating** the work. The more frequently you commit offsets, the less duplicates you will see in a crash.

In the examples thus far, we have assumed that the automatic commit policy is enabled. When the setting **enable.auto.commit** is set to true (which is the default), the consumer automatically triggers offset commits periodically according to the interval configured with “auto.commit.interval.ms.” By reducing the commit interval, you can limit the amount of re-processing the consumer must do in the event of a crash.

**commit offset manually**

The commit API itself is trivial to use, but the most important point is how it is integrated into the poll loop. The following examples therefore include the full poll loop with the commit details in bold. The easiest way to handle commits manually is with the synchronous commit API:

```java
try {
  while (running) {
    ConsumerRecords<String, String> records = consumer.poll(1000);
    for (ConsumerRecord<String, String> record : records)
      System.out.println(record.offset() + ": " + record.value());

    try {
      consumer.commitSync();
    } catch (CommitFailedException e) {
      // application specific failure handling
    }
  }
} finally {
  consumer.close();
}
```

Using the _commitSync_ API with no arguments commits the offsets returned in the last call to poll. This call will block indefinitely until either the commit succeeds or it fails with an unrecoverable error. The main error you need to worry about occurs when message processing takes longer than the session timeout. When this happens, the coordinator kicks the consumer out of the group, which results in a thrown _CommitFailedException_. Your application should handle this error by trying to rollback any changes caused by the consumed messages since the last successfully committed offset.

Typically you should ensure that offset are committed only after the messages have been successfully processed. If the consumer crashes before a commit can be sent, then messages will have to be processed again. If the commit policy guarantees that the last committed offset never gets ahead of the current position, then you have “**at least once**” delivery semantics.

By using the commit API, however, you have much finer control over how much duplicate processing you are willing to accept. In the most extreme case, you could commit **offsets after every message is processed**, as in the following example:

```java
    try {
      for (ConsumerRecord<String, String> record : records) {
        System.out.println(record.offset() + ": " + record.value());
        consumer.commitSync(Collections.singletonMap(record.partition(), new OffsetAndMetadata(record.offset() + 1)));
      }
    } catch (CommitFailedException e) {
      // application specific failure handling
    }
```
In this example, we’ve passed the explicit offset we want to commit in the call to commitSync. The committed offset should always be the offset of the next message that your application will read. When **commitSync** is called with no arguments, the consumer commits the last offsets (plus one) that were returned to the application, but we can’t use that here since that since it would allow the committed position to get ahead of our actual progress. 

Instead of committing on every message received, a more reasonably policy might be to commit offsets as you finish handling the messages from each partition. The ConsumerRecords collection provides access to the set of partitions contained in it and to the messages for each partition. The example below demonstrates this policy.

```java
try {
  while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
    for (TopicPartition partition : records.partitions()) {
      List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
      for (ConsumerRecord<String, String> record : partitionRecords)
        System.out.println(record.offset() + ": " + record.value());

      long lastoffset = partitionRecords.get(partitionRecords.size() - 1).offset();
      consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(lastoffset + 1)));
    }
  }
} finally {
  consumer.close();
} 
```

The examples so far have focused on the synchronous commit API, but the consumer also exposes an asynchronous API, commitAsync. Using asynchronous commits will generally give you higher throughput since your application can begin processing the next batch of messages before the commit returns. The tradeoff is that you may only find out later that the commit failed. The example below shows the basic usage: 

```java
try {
  while (running) {
    ConsumerRecords<String, String> records = consumer.poll(1000);
    for (ConsumerRecord<String, String> record : records)
      System.out.println(record.offset() + ": " + record.value());

    consumer.commitAsync(new OffsetCommitCallback() {
      @Override
      public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, 
                             Exception exception) {
        if (exception != null) {
          // application specific failure handling
        }
      }
    });
  }
} finally {
  consumer.close();
} 
```

### Consumer details

### KafkaConsumer

```java
public class KafkaConsumer<K, V> implements Consumer<K, V> {
    private String clientId;
    private final ConsumerCoordinator coordinator;
    private final Fetcher<K, V> fetcher;
    private final ConsumerNetworkClient client;
    
    public KafkaConsumer() {
        clientId = config.getString(ConsumerConfig.CLIENT_ID_CONFIG);
        if (clientId.length() <= 0)
            clientId = "consumer-" + CONSUMER_CLIENT_ID_SEQUENCE.getAndIncrement();
        
        NetworkClient netClient = new NetworkClient(
             new Selector(config.getLong(ConsumerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), metrics, time, metricGrpPrefix, metricsTags, channelBuilder),
             this.metadata,
             clientId,
             100, // a fixed large enough value will suffice
             config.getLong(ConsumerConfig.RECONNECT_BACKOFF_MS_CONFIG),
             config.getInt(ConsumerConfig.SEND_BUFFER_CONFIG),
             config.getInt(ConsumerConfig.RECEIVE_BUFFER_CONFIG),
             config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG), time);
        
        this.client = new ConsumerNetworkClient(netClient, metadata, time, retryBackoffMs);
        
        this.client = new ConsumerNetworkClient(netClient, metadata, time, retryBackoffMs);
        
        OffsetResetStrategy offsetResetStrategy = OffsetResetStrategy.valueOf(config.getString(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG).toUpperCase());
        
         this.coordinator = new ConsumerCoordinator(this.client,
            config.getString(ConsumerConfig.GROUP_ID_CONFIG),
            config.getInt(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG),
            config.getInt(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG),
            assignors,
            this.metadata,
            this.subscriptions,
            metrics,
            metricGrpPrefix,
            metricsTags,
            this.time,
            retryBackoffMs,
            new ConsumerCoordinator.DefaultOffsetCommitCallback(),
            config.getBoolean(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG),
            config.getLong(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG));

         this.fetcher = new Fetcher<>(this.client,
            config.getInt(ConsumerConfig.FETCH_MIN_BYTES_CONFIG),
            config.getInt(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG),
            config.getInt(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG),
            config.getBoolean(ConsumerConfig.CHECK_CRCS_CONFIG),
            this.keyDeserializer,
            this.valueDeserializer,
            this.metadata,
            this.subscriptions,
            metrics,
            metricGrpPrefix,
            metricsTags,
            this.time,
            this.retryBackoffMs);
         
    }
    
        public ConsumerRecords<K, V> poll(long timeout) {
            acquire();
            try {
                if (timeout < 0)
                    throw new IllegalArgumentException("Timeout must not be negative");
    
                // poll for new data until the timeout expires
                long start = time.milliseconds();
                long remaining = timeout;
                do {
                    Map<TopicPartition, List<ConsumerRecord<K, V>>> records = pollOnce(remaining);
                    if (!records.isEmpty()) {
                        // before returning the fetched records, we can send off the next round of fetches
                        // and avoid block waiting for their responses to enable pipelining while the user
                        // is handling the fetched records.
                        //
                        // NOTE that we use quickPoll() in this case which disables wakeups and delayed
                        // task execution since the consumed positions has already been updated and we
                        // must return these records to users to process before being interrupted or
                        // auto-committing offsets
                        fetcher.initFetches(metadata.fetch());
                        client.quickPoll();
                        return new ConsumerRecords<>(records);
                    }
    
                    long elapsed = time.milliseconds() - start;
                    remaining = timeout - elapsed;
                } while (remaining > 0);
    
                return ConsumerRecords.empty();
            } finally {
                release();
            }
        }


        private Map<TopicPartition, List<ConsumerRecord<K, V>>> pollOnce(long timeout) {
                // TODO: Sub-requests should take into account the poll timeout (KAFKA-1894)
                coordinator.ensureCoordinatorKnown();
        
                // ensure we have partitions assigned if we expect to
                if (subscriptions.partitionsAutoAssigned())
                    coordinator.ensurePartitionAssignment();
        
                // fetch positions if we have partitions we're subscribed to that we
                // don't know the offset for
                if (!subscriptions.hasAllFetchPositions())
                    updateFetchPositions(this.subscriptions.missingFetchPositions());
        
                // init any new fetches (won't resend pending fetches)
                Cluster cluster = this.metadata.fetch();
                Map<TopicPartition, List<ConsumerRecord<K, V>>> records = fetcher.fetchedRecords();
        
                // if data is available already, e.g. from a previous network client poll() call to commit,
                // then just return it immediately
                // 不管有多少剩余么 ?
                if (!records.isEmpty()) {
                    return records;
                }
        
                fetcher.initFetches(cluster);
                client.poll(timeout);
                return fetcher.fetchedRecords();
            }
}
```

上面的实现是 kafka 0.9 redesign [2]

**何为coordinator?**

在0.9以前的client api中，consumer是要依赖Zookeeper的。因为同一个consumer group中的所有consumer需要进行协同，进行下面所讲的rebalance。

但是因为zookeeper的“herd”与“split brain”，导致一个group里面，不同的consumer拥有了同一个partition，进而会引起消息的消费错乱。为此，在0.9中，不再用zookeeper，而是Kafka集群本身来进行consumer之间的同步。下面引自kafka设计的原文：

The current version of the high level consumer suffers from herd and split brain problems, where multiple consumers in a group run a distributed algorithm to agree on the same partition ownership decision. Due to different view of the zookeeper data, they run into conflicts that makes the rebalancing attempt fail. But there is no way for a consumer to verify if a rebalancing operation completed successfully on the entire group. This also leads to some potential bugs in the rebalancing logic, for example, [issue](https://issues.apache.org/jira/browse/KAFKA-242)

**为什么在一个group内部，1个parition只能被1个consumer拥有？**

我们知道，对于属于不同consumer group的consumers，可以消费同1个partition，从而实现Pub/Sub模式。

但是在一个group内部，是不允许多个consumer消费同一个partition的，这也就意味着，对于1个topic，1个group来说， 其partition数目 >= consumer个数。

比如对于1个topic，有4个partition，那么在一个group内部，最多只能有4个consumer。你加入更多的consumer，它们也不会分配到partition。

简单来说，一个是因为这样做就没办法保证同1个partition消息的时序；另1方面，Kafka的服务器，是每个topic的每个partition的每个consumer group对应1个offset，即(topic, partition, consumer_group_id) – offset。如果多个consumer并行消费同1个partition，那offset的confirm就会出问题。

**coordinator协议 / partition分配问题**

给定一个topic，有4个partition: p0, p1, p2, p3， 一个group有3个consumer: c0, c1, c2。那么，如果按范围分配策略，分配结果是： c0: p0, c1: p1, c2: p2, p3 
如果按轮询分配策略： c0: p1, p3, c1: p1, c2: p2

**3 步分配过程**

步骤1：对于每1个consumer group，Kafka集群为其从broker集群中选择一个broker作为其coordinator。因此，第1步就是找到这个coordinator。

```java
    private RequestFuture<Void> sendGroupMetadataRequest() {
        // initiate the group metadata request
        // find a node to ask about the coordinator
        Node node = this.client.leastLoadedNode();
        if (node == null) {
            // TODO: If there are no brokers left, perhaps we should use the bootstrap set
            // from configuration?
            return RequestFuture.noBrokersAvailable();
        } else {
            // create a group  metadata request
            GroupCoordinatorRequest metadataRequest = new GroupCoordinatorRequest(this.groupId);
            return client.send(node, ApiKeys.GROUP_COORDINATOR, metadataRequest)
                    .compose(new RequestFutureAdapter<ClientResponse, Void>() {
                        @Override
                        public void onSuccess(ClientResponse response, RequestFuture<Void> future) {
                            handleGroupMetadataResponse(response, future);
                        }
                    });
        }
    }

    private void handleGroupMetadataResponse(ClientResponse resp, RequestFuture<Void> future) {
        
        if (!coordinatorUnknown()) {
            // We already found the coordinator, so ignore the request
            future.complete(null);
        } else {
            GroupCoordinatorResponse groupCoordinatorResponse = new GroupCoordinatorResponse(resp.responseBody());
            // use MAX_VALUE - node.id as the coordinator id to mimic separate connections
            // for the coordinator in the underlying network client layer
            // TODO: this needs to be better handled in KAFKA-1935
            short errorCode = groupCoordinatorResponse.errorCode();
            if (errorCode == Errors.NONE.code()) {
                this.coordinator = new Node(Integer.MAX_VALUE - groupCoordinatorResponse.node().id(),
                        groupCoordinatorResponse.node().host(),
                        groupCoordinatorResponse.node().port());

                client.tryConnect(coordinator);

                // start sending heartbeats only if we have a valid generation
                if (generation > 0)
                    heartbeatTask.reset();
                future.complete(null);
            } else if (errorCode == Errors.GROUP_AUTHORIZATION_FAILED.code()) {
                future.raise(new GroupAuthorizationException(groupId));
            } else {
                future.raise(Errors.forCode(errorCode));
            }
        }
    }
```

步骤2：找到coordinator之后，发送JoinGroup请求

```java
    /**
     * Ensure that we have a valid partition assignment from the coordinator.
     */
    public void ensurePartitionAssignment() {
        if (subscriptions.partitionsAutoAssigned())
            ensureActiveGroup();
    }
    
    public void ensureActiveGroup() {
        RequestFuture<ByteBuffer> future = performGroupJoin();
        client.poll(future);
    }
    
    private RequestFuture<ByteBuffer> performGroupJoin() {
        JoinGroupRequest request = new JoinGroupRequest(
            groupId,
            this.sessionTimeoutMs,
            this.memberId,
            protocolType(),
            metadata());
        
            // create the request for the coordinator
            log.debug("Issuing request ({}: {}) to coordinator {}", ApiKeys.JOIN_GROUP, request, this.coordinator.id());
            return client.send(coordinator, ApiKeys.JOIN_GROUP, request)
                .compose(new JoinGroupResponseHandler());
    }
    
    private class JoinGroupResponseHandler extends CoordinatorResponseHandler<JoinGroupResponse, ByteBuffer> {
        public void handle(JoinGroupResponse joinResponse, RequestFuture<ByteBuffer> future) {
            short errorCode = joinResponse.errorCode();
            if (errorCode == Errors.NONE.code()) {
                AbstractCoordinator.this.memberId = joinResponse.memberId();
                AbstractCoordinator.this.generation = joinResponse.generationId();
                AbstractCoordinator.this.rejoinNeeded = false;
                AbstractCoordinator.this.protocol = joinResponse.groupProtocol();
                sensors.joinLatency.record(response.requestLatencyMs());
                if (joinResponse.isLeader()) {
                    onJoinLeader(joinResponse).chain(future);
                } else {
                    onJoinFollower().chain(future);
                }
            }
        }
    }
    
    private RequestFuture<ByteBuffer> onJoinFollower() {
            // send follower's sync group with an empty assignment
            SyncGroupRequest request = new SyncGroupRequest(groupId, generation,
                    memberId, Collections.<String, ByteBuffer>emptyMap());
            log.debug("Issuing follower SyncGroup ({}: {}) to coordinator {}", ApiKeys.SYNC_GROUP, request, this.coordinator.id());
            return sendSyncGroupRequest(request);
    }
    
    private RequestFuture<ByteBuffer> onJoinLeader(JoinGroupResponse joinResponse) {
            try {
                // perform the leader synchronization and send back the assignment for the group
                Map<String, ByteBuffer> groupAssignment = performAssignment(joinResponse.leaderId(), joinResponse.groupProtocol(),
                        joinResponse.members());
    
                SyncGroupRequest request = new SyncGroupRequest(groupId, generation, memberId, groupAssignment);
                log.debug("Issuing leader SyncGroup ({}: {}) to coordinator {}", ApiKeys.SYNC_GROUP, request, this.coordinator.id());
                return sendSyncGroupRequest(request);
            } catch (RuntimeException e) {
                return RequestFuture.failure(e);
            }
    }
```

这里注意一点，Coordinator 有 client 和 server 两个 instance, consumer 本身也要维持一个 coordinator client, 它的定义在 kafka/clients 包下

此外，JoinGroup 和下面的 SyncGroup 都会发给 broker 的 coordinator, heartBeat 也会发给 server 的 coordinator。

步骤3：JoinGroup返回之后，发送SyncGroup，得到自己所分配到的partition

上一步的代码中已经包括发送 JoinGroup 流程，下面是发送 SyncGroup 后的回调函数

```java
    private class SyncGroupRequestHandler extends CoordinatorResponseHandler<SyncGroupResponse, ByteBuffer> {

        @Override
        public SyncGroupResponse parse(ClientResponse response) {
            return new SyncGroupResponse(response.responseBody());
        }

        @Override
        public void handle(SyncGroupResponse syncResponse,
                           RequestFuture<ByteBuffer> future) {
            Errors error = Errors.forCode(syncResponse.errorCode());
            if (error == Errors.NONE) {
                log.debug("Received successful sync group response for group {}: {}", groupId, syncResponse.toStruct());
                sensors.syncLatency.record(response.requestLatencyMs());
                future.complete(syncResponse.memberAssignment());
            }
        }
    }
```
partition的分配策略和分配结果其实是由client决定的，而不是由coordinator决定的。什么意思呢？在第2步，所有consumer都往coordinator发送JoinGroup消息之后，coordinator会指定其中一个consumer作为leader，其他consumer作为follower。然后由这个leader进行partition分配。然后在第3步，leader通过SyncGroup消息，把分配结果发给coordinator，其他consumer也发送SyncGroup消息，获得这个分配结果。

为什么要在consumer中选一个leader出来，进行分配，而不是由coordinator直接分配呢？关于这个, Kafka的官方文档有详细的分析。其中一个重要原因是为了灵活性：如果让server分配，一旦需要新的分配策略，server集群要重新部署，这对于已经在线上运行的集群来说，代价是很大的；而让client分配，server集群就不需要重新部署了。

如果分配出现了 conflict, coordinator 会返回异常

### rebalance机制

所谓rebalance，就是在某些条件下，partition要在consumer中重新分配。那哪些条件下，会触发rebalance呢？ 

- 条件1：有新的consumer加入 
- 条件2：旧的consumer挂了 
- 条件3：coordinator挂了，集群选举出新的coordinator 
- 条件4：topic的partition新加 
- 条件5：consumer调用unsubscrible()，取消topic的订阅



server 负责管理分配策略的问题在于

1. First is just a matter of convenience. New assignment strategies cannot be deployed to the server without updating configuration and restarting the cluster. It can be a significant operational undertaking just to provide the capability to do this. 
2. Different assignment strategies have different validation requirements. For example, with a redundant partitioning scheme, a single partition can be assigned to multiple consumers. This limits the ability of the coordinator to validate assignments, which is one of the main reasons for having the coordinator do the assignment in the first place.

**Official documentation: Protocol**

**Phase1: Joining the group**

The purpose of the initial phase is to set the active members of the group. This protocol has similar semantics as in the initial consumer rewrite design. After finding the coordinator for the group, each member sends a JoinGroup request containing member-specific metadata. The join group request will park at the coordinator **until all expected members** have sent their own join group requests ("expected" in this case means **all members that were part of the previous generation**). Once they have done so, the coordinator **randomly selects** a leader from the group and sends JoinGroup responses to all the pending requests.

The JoinGroup request contains an array with the group protocols that it supports along with member-specific metadata. This is basically used to ensure compatibility of group member metadata within the group. The coordinator chooses a protocol which is supported by all members of the group and returns it in the respective JoinGroup responses. If a member joins and doesn't support any of the protocols used by the rest of the group, then it will be rejected. This mechanism provides a way to update protocol metadata to a new format in a rolling upgrade scenario. The newer version will provide metadata for the new protocol and for the old protocol, and the coordinator will choose the old protocol until all members have been upgraded. 


The JoinGroup response includes an array for the members of the group along with their metadata. **This is only populated for the leader to reduce the overall overhead of the protocol**; for other members, it will be empty. The is used by the leader to prepare member state for phase 2. In the case of the consumer, this allows the leader to collect the subscriptions from all members and set the partition assignment. The member metadata returned in the join group response corresponds to the respective metadata provided in the join group request for the group protocol chosen by the coordinator.

**Phase 2: Synchronizing Group State**

Once the group members have been stabilized by the completion of phase 1, **the active leader must propagate state to the other members in the group**. This is used in the new consumer protocol to **set partition assignments**. Similar to phase 1, all members send SyncGroup requests to the coordinator. Once group state has been provided by the leader, the coordinator forwards each member's state respectively in the SyncGroup response. The message format is provided below:

The leader sets member states in the **GroupState** field. For followers, this array must be empty. Once the coordinator has received the group state from the leader, it can unpack each member's state and send it in the MemberState field of the SyncGroup response.

**Coordinator State Machine**

**Down:** There are no active members and group state has been cleaned up.

**Initialize:** In this state, the coordinator reads group data from Zookeeper (or some other storage) in order to transition groups from failed coordinators. Any heartbeat or join group requests are returned with an error indicating that the coordinator is not ready yet.

**Stable:** In this state, the coordinator either has an active generation or has no members and is awaiting the first JoinGroup. Heartbeats are accepted from members in this state and are used to keep group members active or to indicate that they need to join the group.

**Joining:** The coordinator has received a JoinGroup request from at least one member and is awaiting JoinGroup requests from the rest of the group. Heartbeats or SyncGroup requests in this state return an error indicating that a rebalance is in progress.

**AwaitingSync:** The join group phase has completed (i.e. all expected members of the group have sent JoinGroup requests) and the coordinator is awaiting group state from the leader. Unexpected coordinator requests return an error indicating that a rebalance is in progress. 

![](/images/posts/kafka/kafka-state-machine.png)

Note that the generation is incremented on successful completion of the first phase (Joining). Before this phase completes, the old generation has an opportunity to do any necessary cleanup work (such as commit offsets in the case of the new consumer). Upon transition to the AwaitSync state, the coordinator begins a timer for each member according to their respective session timeouts. If the timeout expires for any member, then the coordinator must trigger a rebalance.

The group leader is responsible for synchronizing state for the group upon completion of the Joining state. If the leader's session timeout expires before the coordinator has received the leader's SyncGroup, then the generation becomes invalid and the members must rejoin. It is also possible for the leader to transmit an error in the SyncGroup request. In this case also, the generation becomes invalid and the error can be propagated to the other members of the group. 

Note that the transition from AwaitSync to Stable occurs only when the **leader's SyncGroup has been received**. It is possible that the SyncGroup from followers may therefore arrive either in the AwaitSync state or in the Stable state. If the former, then the coordinator will **park the request until the SyncGroup from the leader has been received** (or its timeout has expired). If the latter, then the coordinator can respond to the SyncGroup request immediately using the leader's synchronized state. Clearly this requires the coordinator to store this state at least for the duration of the max session timeout in the group. It is also be possible that the member fails before collecting its state: in this case, the member's session timeout will expire and the group will rebalance.

_**Consumer Embedded Protocol**_

Above we outlined the generalized JoinGroup protocol that the consumer will use. Next we show how we will implement consumer semantics on top of this protocol. Other use cases for the join group protocol would be implemented similarly.

The two phases of the group protocol correspond to subscription and assignment for the new consumer. Each member of the group submits their subscription as member metadata. The **leader** of the group collects all subscription in its JoinGroup response and sends the assignment as member state in SyncGroup. There are several advantages to having a single assignor:

1. Since the leader makes the assignment for the full group, it is the single source of truth for the metadata used in its decision making. This avoids the need to synchronize metadata among all members that is required in a multi-assignor approach. 
2. The leader of the group can enforce its own policy for controlling the rate of rebalancing. It doesn't have to rebalance after every metadata change, but can "batch" changes together to reduce the impact of metadata churn.
3. The leader is the only member that needs to receive the metadata from all members of the group. This reduces the overhead of the protocol.

The group protocol used by the consumer in the JoinGroup request corresponds to the assignment strategy that the leader will use to determine partition assignment. This allows the consumer to upgrade from one assignment strategy to another without downtime. The metadata corresponding to the assignment strategy can be strategy-specific, but generally it will include the group subscriptions for the member. The state returned to members in the SyncGroup will include the partitions assigned to that member.

For all assignment strategies, group members provide their subscriptions as an array of strings. This subscription can either be a list of topics or regular expressions (TODO: do we need distinguisher field to tell the difference? how about regex compatibility?). Partition assignments are provided in the SyncGroup response as an array of topics and partitions. The protocol supports custom data in both the subscription and assignment as a generic array of bytes to allow for custom assignor implementations. For example, a rack-aware assignor will generally need to propagate the rackId of each member to the leader in its subscription so that it can take it into account for assignment.  


### offset management



### Reference:

[Link1](http://blog.csdn.net/chunlongyu/article/details/52791874)
[Official Blog](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Client-side+Assignment+Proposal)