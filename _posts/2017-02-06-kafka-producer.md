---
layout: post
title: Kafka producer
categories: [kafka]
keywords: kafka producer
---

Producer有同步发送和异步发送2种策略。在以前的Kafka client api实现中，同步和异步是分开实现的。而在0.9中，同步发送其实是通过异步发送间接实现，其接口如下：

```java
public class KafkaProducer<K, V> implements Producer<K, V> {
...
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback)  //异步发送接口
     {
     ...
     }
}
```

要实现同步发送，只要在拿到返回的Future对象之后，直接调用get()就可以了。

**基本思路**

从上图我们可以看出，异步发送的基本思路就是：send的时候，KafkaProducer把消息放到本地的消息队列RecordAccumulator，然后一个后台线程Sender不断循环，把消息发给Kafka集群。

要实现这个，还得有一个前提条件：就是KafkaProducer/Sender都需要获取集群的配置信息Metadata。所谓Metadata，也就是在上一篇所讲的，Topic/Partion与broker的映射关系：每一个Topic的每一个Partion，得知道其对应的broker列表是什么，其中leader是谁，follower是谁。

**Metadata的数据结构**

下面代码列举了Metadata的主要数据结构：一个Cluster对象 + 1堆状态变量。前者记录了集群的配置信息，后者用于控制Metadata的更新策略。

```java
public final class Metadata {
...
    private final long refreshBackoffMs;  //更新失败的情况下，下 1 次更新的补偿时间（这个变量在代码中意义不是太大）
    private final long metadataExpireMs; //关键值：每隔多久，更新一次。缺省是600*1000，也就是10分种
    private int version;         //每更新成功1次，version递增1。这个变量主要用于在while循环，wait的时候，作为循环判断条件
    private long lastRefreshMs;  //上一次更新时间（也包含更新失败的情况）
    private long lastSuccessfulRefreshMs; //上一次成功更新的时间（如果每次都成功的话，则2者相等。否则，lastSuccessulRefreshMs < lastRefreshMs)
    private Cluster cluster;   //集群配置信息
    private boolean needUpdate;  //是否强制刷新
、
  ...
}

public final class Cluster {
...
    private final List<Node> nodes;   //Node也就是Broker
    private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;  //Topic/Partion和broker list的映射关系
    private final Map<String, List<PartitionInfo>> partitionsByTopic;
    private final Map<String, List<PartitionInfo>> availablePartitionsByTopic;
    private final Map<Integer, List<PartitionInfo>> partitionsByNode;
    private final Map<Integer, Node> nodesById;
}

public class PartitionInfo {
    private final String topic;
    private final int partition;
    private final Node leader;
    private final Node[] replicas;
    private final Node[] inSyncReplicas;
}
```

**producer 读取 Metadata**

下面是send函数的源码，可以看到，在send之前，会先读取metadata。如果metadata读不到，会一直阻塞在那，直到超时，抛出TimeoutException

```java
//KafkaProducer
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        try {
     long waitedOnMetadataMs = waitOnMetadata(record.topic(), this.maxBlockTimeMs);  //拿不到topic的配置信息，会一直阻塞在这，直到抛异常

     ... //拿到了，执行下面的send逻辑
     } catch()
     {}
 }

//KafkaProducer
    private long waitOnMetadata(String topic, long maxWaitMs) throws InterruptedException {
        if (!this.metadata.containsTopic(topic))
            this.metadata.add(topic);

        if (metadata.fetch().partitionsForTopic(topic) != null)
            return 0;   //取到topic的配置信息，直接返回

        long begin = time.milliseconds();
        long remainingWaitMs = maxWaitMs;
        while (metadata.fetch().partitionsForTopic(topic) == null) { //取不到topic的配置信息，一直死循环wait，直到超时，抛TimeoutException
            log.trace("Requesting metadata update for topic {}.", topic);
            int version = metadata.requestUpdate(); //把needUpdate置为true
            sender.wakeup(); //唤起sender

            metadata.awaitUpdate(version, remainingWaitMs); //metadata的关键函数
            long elapsed = time.milliseconds() - begin;
            if (elapsed >= maxWaitMs)
                throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
            if (metadata.fetch().unauthorizedTopics().contains(topic))
                throw new TopicAuthorizationException(topic);
            remainingWaitMs = maxWaitMs - elapsed;
        }
        return time.milliseconds() - begin;
    }

//Metadata
    public synchronized void awaitUpdate(final int lastVersion, final long maxWaitMs) throws InterruptedException {
        if (maxWaitMs < 0) {
            throw new IllegalArgumentException("Max time to wait for metadata updates should not be < 0 milli seconds");
        }
        long begin = System.currentTimeMillis();
        long remainingWaitMs = maxWaitMs;
        while (this.version <= lastVersion) {  //当Sender成功更新meatadata之后，version加1。否则会循环，一直wait
            if (remainingWaitMs != 0
                wait(remainingWaitMs);  //线程的wait机制，wait和synchronized的配合使用
            long elapsed = System.currentTimeMillis() - begin;
            if (elapsed >= maxWaitMs)  //wait时间超出了最长等待时间
                throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
            remainingWaitMs = maxWaitMs - elapsed;
        }
}
```

总结：从上面代码可以看出，producer wait metadata的时候，有2个条件： 
（1） while (metadata.fetch().partitionsForTopic(topic) == null) 
（2）while (this.version <= lastVersion)

有wait就会有notify，notify在Sender更新Metadata的时候发出。

**Sender 线程 poll metadata**

```java
    public void run(long now) {
        Cluster cluster = metadata.fetch();
。。。
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);   //遍历消息队列中所有的消息，找出对应的，已经ready的Node

        if (result.unknownLeadersExist)  //如果一个ready的node都没有，请求更新metadata
            this.metadata.requestUpdate();

  。。。

     //client的2个关键函数，一个发送ClientRequest，一个接收ClientResponse。底层调用的是NIO的poll。关于nio, 后面会详细介绍
        for (ClientRequest request : requests)
            client.send(request, now);

        this.client.poll(pollTimeout, now);
    }

//NetworkClient
    public List<ClientResponse> poll(long timeout, long now) {
        long metadataTimeout = metadataUpdater.maybeUpdate(now); //关键点：每次poll的时候判断是否要更新metadata

        try {
            this.selector.poll(Utils.min(timeout, metadataTimeout, requestTimeoutMs));
        } catch (IOException e) {
            log.error("Unexpected error during I/O", e);
        }

        // process completed actions
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
        handleCompletedSends(responses, updatedNow);
        handleCompletedReceives(responses, updatedNow);   //在返回的handler中，会处理metadata的更新
        handleDisconnections(responses, updatedNow);
        handleConnections();
        handleTimedOutRequests(responses, updatedNow);

        // invoke callbacks
        for (ClientResponse response : responses) {
            if (response.request().hasCallback()) {
                try {
                    response.request().callback().onComplete(response);
                } catch (Exception e) {
                    log.error("Uncaught error in request completion:", e);
                }
            }
        }

        return responses;
    }
```

**Metadata的2种更新机制**

(1)周期性的更新: 每隔一段时间更新一次，这个通过 Metadata的lastRefreshMs, lastSuccessfulRefreshMs 这2个字段来实现

对应的ProducerConfig配置项为： 
metadata.max.age.ms //缺省300000，即10分钟1次

(2) 失效检测，强制更新：检查到metadata失效以后，调用metadata.requestUpdate()强制更新。 requestUpdate()函数里面其实什么都没做，就是把needUpdate置成了false

每次poll的时候，都检查这2种更新机制，达到了，就触发更新。

那如何判定Metadata失效了呢？这个在代码中很分散，有很多地方，会判定Metadata失效。

**Metadata失效检测**

- 条件1：initConnect的时候
- 条件2：poll里面IO的时候，连接断掉了
- 条件3：有请求超时
- 条件4：发消息的时候，有partition的leader没找到
- 条件5：返回的response和请求对不上的时候
  
### Kafka networking

![](/images/posts/kafka/kafka-networking.png)
  
KakfaChannel基本是对SocketChannel的封装，只是这个中间多个一个间接层：TransportLayer，为了封装普通和加密的Channel；

Send/NetworkReceive是对ByteBuffer的封装，表示一次请求的数据包；

Kafka的Selector封装了NIO的Selector，内含一个NIO Selector对象。

**Kafka Selector实现思路**

```
private final Map<String, KafkaChannel> channels;
```

NetworkClient的send()函数，调用了selector.send(Send send)， 但这个时候数据并没有真的发送出去，只是暂存在了selector内部相对应的channel里面。下面看代码：

```java
//Selector
    public void send(Send send) {
        KafkaChannel channel = channelOrFail(send.destination());  //找到数据包相对应的connection
        try {
            channel.setSend(send);  //暂存在这个connection(channel)里面
        } catch (CancelledKeyException e) {
            this.failedSends.add(send.destination());
            close(channel);
        }
    }

//KafkaChannel
    public void setSend(Send send) {
        if (this.send != null)  //关键点：当前的没有发出去之前，不能暂存下1个！！！关于这个，后面还要详细分析
            throw new IllegalStateException("Attempt to begin a send operation with prior send operation still in progress.");
        this.send = send;   //暂存这个数据包
        this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
    }

public class KafkaChannel {
    private final String id;
    private final TransportLayer transportLayer;
    private final Authenticator authenticator;
    private final int maxReceiveSize;
    private NetworkReceive receive;
    private Send send;   //关键点：1个channel一次只能存放1个数据包，在当前的send数据包没有完整发出去之前，不能存放下一个
    ...
}
```

**核心原理之3 － 消息时序保证**

在InFlightRequests中，存放了所有发出去，但是response还没有回来的request。request发出去的时候，入对；response回来，就把相对应的request出对。

这个有个关键点：我们注意到request与response的配对，在这里是用队列表达的，而不是Map。用队列的入队，出队，完成2者的匹配。要实现这个，服务器就必须要保证消息的时序：即在一个socket上面，假如发出去的reqeust是0, 1, 2，那返回的response的顺序也必须是0, 1, 2。

但是服务器是1 + N + M模型，所有的请求进入一个requestQueue，然后是多线程并行处理的。那它如何保证消息的时序呢？

答案是mute/unmute机制：每当一个channel上面接收到一个request，这个channel就会被mute，然后等response返回之后，才会再unmute。这样就保证了同1个连接上面，同时只会有1个请求被处理。


### RecordAccumulator队列分析

**Batch发送**

在以前的kafka client中，每条消息称为 “Message”，而在Java版client中，称之为”Record”，同时又因为有批量发送累积功能，所以称之为RecordAccumulator.

RecordAccumulator最大的一个特性就是batch消息，扔到队列中的多个消息，可能组成一个RecordBatch，然后由Sender一次性发送出去。

**每个TopicPartition一个队列**

下面是RecordAccumulator的内部结构，可以看到，每个TopicPartition对应一个消息队列，只有同一个TopicPartition的消息，才可能被batch。

```
public final class RecordAccumulator {
    private final ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches;

   ...
}
```

**batch的策略**

```
//KafkaProducer
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        try {
            // first make sure the metadata for the topic is available
            long waitedOnMetadataMs = waitOnMetadata(record.topic(), this.maxBlockTimeMs);

            ...

            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, serializedKey, serializedValue, callback, remainingWaitMs);   //核心函数：把消息放入队列

            if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
                this.sender.wakeup();
            }
            return result.future;
```

从上面代码可以看到，batch逻辑，都在accumulator.append函数里面：


```java
public RecordAppendResult append(TopicPartition tp, byte[] key, byte[] value, Callback callback, long maxTimeToBlock) throws InterruptedException {
        appendsInProgress.incrementAndGet();
        try {
            if (closed)
                throw new IllegalStateException("Cannot send after the producer is closed.");
            Deque<RecordBatch> dq = dequeFor(tp);  //找到该topicPartiton对应的消息队列
            synchronized (dq) {
                RecordBatch last = dq.peekLast(); //拿出队列的最后1个元素
                if (last != null) {  
                    FutureRecordMetadata future = last.tryAppend(key, value, callback, time.milliseconds()); //最后一个元素, 即RecordBatch不为空，把该Record加入该RecordBatch
                    if (future != null)
                        return new RecordAppendResult(future, dq.size() > 1 || last.records.isFull(), false);
                }
            }

            int size = Math.max(this.batchSize, Records.LOG_OVERHEAD + Record.recordSize(key, value));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            
            ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
            
            synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");
                RecordBatch last = dq.peekLast();
                if (last != null) {
                    FutureRecordMetadata future = last.tryAppend(key, value, callback, time.milliseconds());
                    if (future != null) {
                        // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                        free.deallocate(buffer);
                        return new RecordAppendResult(future, dq.size() > 1 || last.records.isFull(), false);
                    }
                }

                //队列里面没有RecordBatch，建一个新的，然后把Record放进去
                MemoryRecords records = MemoryRecords.emptyRecords(buffer, compression, this.batchSize);
                RecordBatch batch = new RecordBatch(tp, records, time.milliseconds());
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(key, value, callback, time.milliseconds()));

                dq.addLast(batch);
                incomplete.add(batch);
                return new RecordAppendResult(future, dq.size() > 1 || batch.records.isFull(), true);
            }
        } finally {
            appendsInProgress.decrementAndGet();
        }
    }

    private Deque<RecordBatch> dequeFor(TopicPartition tp) {
        Deque<RecordBatch> d = this.batches.get(tp);
        if (d != null)
            return d;
        d = new ArrayDeque<>();
        Deque<RecordBatch> previous = this.batches.putIfAbsent(tp, d);
        if (previous == null)
            return d;
        else
            return previous;
    }
```

从上面代码我们可以看出Batch的策略： 

1. 如果是同步发送，每次去队列取，RecordBatch都会为空。这个时候，消息就不会batch，一个Record形成一个RecordBatch

2. Producer 入队速率 < Sender出队速率 && lingerMs = 0 ，消息也不会被 batch

3. Producer 入队速率 > Sender出对速率， 消息会被 batch

4. lingerMs > 0，这个时候Sender会等待，直到lingerMs > 0 或者 队列满了，或者超过了一个RecordBatch的最大值，就会发送。这个逻辑在RecordAccumulator的ready函数里面。

```java
    public ReadyCheckResult ready(Cluster cluster, long nowMs) {
        Set<Node> readyNodes = new HashSet<Node>();
        long nextReadyCheckDelayMs = Long.MAX_VALUE;
        boolean unknownLeadersExist = false;

        boolean exhausted = this.free.queued() > 0;
        for (Map.Entry<TopicPartition, Deque<RecordBatch>> entry : this.batches.entrySet()) {
            TopicPartition part = entry.getKey();
            Deque<RecordBatch> deque = entry.getValue();

            Node leader = cluster.leaderFor(part);
            if (leader == null) {
                unknownLeadersExist = true;
            } else if (!readyNodes.contains(leader)) {
                synchronized (deque) {
                    RecordBatch batch = deque.peekFirst();
                    if (batch != null) {
                        boolean backingOff = batch.attempts > 0 && batch.lastAttemptMs + retryBackoffMs > nowMs;
                        long waitedTimeMs = nowMs - batch.lastAttemptMs;
                        long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                        long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                        boolean full = deque.size() > 1 || batch.records.isFull();
                        boolean expired = waitedTimeMs >= timeToWaitMs;
                        boolean sendable = full || expired || exhausted || closed || flushInProgress();  //关键的一句话
                        if (sendable && !backingOff) {
                            readyNodes.add(leader);
                        } else {

                            nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                        }
                    }
                }
            }
        }

        return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeadersExist);
    }
```

**为什么是Deque？**

在上面我们看到，消息队列用的是一个“双端队列“，而不是普通的队列。 
一端生产，一端消费，用一个普通的队列不就可以吗，为什么要“双端“呢？

这其实是为了处理“发送失败，重试“的问题：当消息发送失败，要重发的时候，需要把消息优先放入队列头部重新发送，这就需要用到双端队列，在头部，而不是尾部加入。

当然，即使如此，该消息发出去的顺序，还是和Producer放进去的顺序不一致了。



**Reference:**

[1](http://blog.csdn.net/chunlongyu/article/details/52622422)