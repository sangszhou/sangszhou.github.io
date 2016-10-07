
![](/https://zqhxuyuan.github.io/2016/02/23/2016-02-23-Kafka-Controller/) 看不太懂

## Replication

### 1) 数据同步

kafka在0.8版本前没有提供Partition的Replication机制，一旦Broker宕机，其上的所有Partition就都无法提供服务，
而Partition又没有备份数据，数据的可用性就大大降低了。所以0.8后提供了Replication机制来保证Broker的failover。
由于Partition有多个副本，为了保证多个副本之间的数据同步，有多种方案：

1. 所有副本之间是无中心结构的，可同时读写数据，需要保证多个副本之间数据的同步
2. 在所有副本中选择一个Leader，生产者和消费者只和Leader副本交互，其他follower副本从Leader同步数据

第一种方案看起来可以对客户端请求进行负载均衡，但是由于要在多个副本之间互相同步数据，数据的一致性和有序性难以保证。

第二种方案看起来客户端连接的节点会少点，而且其他副本同步数据可能没有那么及时，但是在正常情况下，
客户端只需要和Leader一个副本通信即可，而其他follower只需要和Leader同步数据。假设有一个Partition有5个副本，
4个follower只需要各自和leader建立一条链路通信，而对于第一种方案，5个副本之间要两两通信，
确保Partition的每个副本的数据都是一致的。所以第一种方案虽然提供了客户端的负载均衡，
但是对于服务端的设计带来比较大的复杂性，而第二种方案虽然限制了客户端只能连接Partition的Leader Replica，
但这种简洁的设计使得服务端更加健壮。

### 2) 同步策略

存在Replication的目的是为了在Leader发生故障时，follower副本能够代替Leader副本继续工作，
即新的Leader必须拥有原来的Leader提交过的所有消息，那么在任何时刻follower要保证和Leader比起来总是最新的，
或者说follower要和leader的数据始终保持同步。

有两种策略保证副本和leader是同步的，primary-backup replication和quorum-based replication。
这两种模式下都会选举一个副本作为leader，其他的副本都作为follower。所有的写请求都会经过leader，
然后leader会将写传播给follower（kafka因为使用pull模式，所以是follower从leader拉取数据，不过总的来说，
目的都是将leader的数据同步给所有的follower）。

使用基于quorum的复制方式（也叫做Majority Vote，少数服从多数），leader需要等待副本集合中大多数的写操作完
成（大多数的概念是指一半以上）。如果一些副本当掉了，副本组的大小也不会发生变化，这就会导致写操作无法写到失败的副本上，
就无法满足半数以上的限制条件。这种模式下，假设有2N+1个副本，在提交之前要确保有N+1个副本复制完消息，
为了保证正确选出新的Leader，失败的副本数不能超过N个。因为在剩下的任意N+1个Replica里，至少有一个Replica包
含有最新的所有消息。这种策略的缺点是：为了保证Leader选举的正常进行，它所能容忍的失败的follower个数比较少，
如果要容忍N个follower挂掉，必须要有2N+1个以上的副本。

使用主备模式复制，leader会等待组（所有的副本组成的集合）中所有副本写成功后才返回应答给客户端。如果其中一个副本当掉了，
leader会将它从组中删除掉，并继续向剩余的副本写数据。一个失败的副本在恢复之后如果能赶上Leader，leader就会允许它重新加
入组中。kafka中这个组的概念就是Leader副本的ISR集合。这个ISR里的所有副本都跟上了Leader，只有ISR里的成员才有
被选为Leader的可能。这种模式下对于N+1个副本，一个Partition能在保证不丢失已经提交的消息的前提下容忍N个副本的失
败（只要有一个副本没有失败，其他失败了都没有关系）。比较Majority Vote和ISR，为了容忍N个副本的失败，两者在
提交前需要等待的副本数量是一样的（ISR中有N+1个副本才可以容忍N个副本失败，而Majority Vote副本总数=2N+1，N+1个
副本成功，才能容忍N个副本失败，所以需要等待的副本都是N+1个。这里的等待一般是将请求发送到N+1个副本后，要等待这些
副本应答后才表示成功提交），但是ISR需要的总的副本数几乎是Majority Vote的一半（假设ISR中所有副本都能跟上Leader，则
一共也之后N+1个副本，而Majority Vote则需要2N+1个副本)

### 3) leader选举

Leader选举本质上是一个分布式锁，有两种方式实现基于ZooKeeper的分布式锁：

1. 节点名称唯一性：多个客户端创建一个节点，只有成功创建节点的客户端才能获得锁
2. 临时顺序节点：所有客户端在某个目录下创建自己的临时顺序节点，只有序号最小的才获得锁

Majority Vote的选举策略和ZooKeeper中的Zab选举是类似的，实际上ZooKeeper内部本身就实现了少数服从多数的选举策略。

kafka中对于Partition的leader副本的选举采用了第一种方法：为Partition分配副本，指定一个ZNode临时节点，
第一个成功创建节点的副本就是Leader节点，其他副本会在这个ZNode节点上注册Watcher监听器，一旦Leader宕机，
对应的临时节点就会被自动删除，这时注册在该节点上的所有Follower都会收到监听器事件，它们都会尝试创建该节点，
只有创建成功的那个follower才会成为Leader（ZooKeeper保证对于一个节点只有一个客户端能创建成功），
其他follower继续重新注册监听事件。

### 4) 副本放置策略

kafka将一个逻辑意义的topic分成多个物理意义上的partition，每个partition是这个topic的一部分消息，
将partition分布在多个brokers上，可以达到负载均衡的目的。除了Partition的分配，每个Partition都有Replcias，
所以也要考虑Replica的分配策略，因为不能把一个Partition的所有Replicas都放在同一台机器上。
一般我们希望将所有的Partition能够均匀地分布在集群中所有的Brokers上。Kafka分配Replica的算法如下：

1. 将所有存活的N个Brokers和待分配的Partition排序
2. 将第i个Partition分配到第(i mod n)个Broker上，这个Partition的第一个Replica存在于这个分配的Broker上，
   并且会作为partition的优先副本
3. 将第i个Partition的第j个Replica分配到第((i + j) mod n)个Broker上

### 9) Follower failure

如果Follower Replica失败超过一定时间后，Leader会将这个失败的follower从ISR中移除（follower没有发送fetch请求）。
由于ISR保存的是所有全部赶得上Leader的follower replicas，失败的follower肯定是赶不上了。
虽然ISR现在少了一个，但是并不会引起的数据的丢失，ISR中剩余的replicas会继续同步数据（只要ISR中
有一个follower，就不会丢失数据，实际上leader replica也是在ISR中的，所以即使所有的follower都挂掉了，
只要leader没有问题，也不会出现问题的）。

当失败的follower恢复时，它首先将自己的日志截断到上次checkpointed时刻的HW（checkpoint是每个broker节点都有的）。
因为checkpoint记录的是所有Partition的hw offset。当follower失败时，checkpoint中关于这个Partition的HW就不会再更新了。
而这个时候存储的HW信息和follower partition replica的offset并不一定是一致的。比如这个follower获取消息比较快，
但是ISR中有其他follower复制消息比较慢，这样Leader并不会很快地更新HW，这个快的follower的hw也不会更
新（leader广播hw给follower）。这种情况下，这个follower日志的offset是比hw要大的。所以在它恢复之后，要
将比hw多的部分截掉，然后继续从leader拉取消息（跟平时一样）。实际上，ISR中的每个follower日志的offset一定
是比hw大的。因为只有ISR中所有follower都复制完消息，leader才会增加hw，而每个Replica复制消息后，都会增加
自己的offset。也就是说有可能有些follower复制完了，而有另外一些follower还没有复制完，那么hw是不会增加的。

Kafka的Replica实现到目前0.9版本为止，分别经历了三个阶段：

1	zookeeper，很多监听器	脑裂，羊群效应，ZK集群负载过重
2	one brain，zk queue	将Replcia改变的状态事件用ZK队列实现
3	state machine，controller, direct rpc	直接RPC更快，状态机统一处理

### 11) zk path & listeners

下表汇总了zk中和partition相关的节点，对于不同版本，路径可能不同，不过要存储的信息都是类似的：

replicas响应partition leader的最长等待时间，若是超过这个时间，**就将replicas列入ISR(in-sync replicas)**，并认为它是死的，
不会再加入管理中

### 12) Broker,Partition和Replica的事件流

Partition的Replica存储在Broker上，并且和ZooKeeper互相交互，主要完成这些工作：

1. Broker启动读取ZK中Partition的所有AR
2. 每个Replica判断Leader是否存在，不存在则进入选举阶段，存在则使自己成为Follower
3. 每个Replica都会在leader上注册监听器，当Leader发生变化时，所有Replica会开始选举Leader
4. 选举Leader时，最先创建leader节点的Replica成为Leader，其他Replica再次成为Follower

v1版本对于每个replica变化都会触发replicaStateChange调用。(AR => All replicas?)

![](/images/posts/kafka/v1_replica_change.png)

v1版本中，由于各种事件严重依赖ZooKeeper，存在着以下问题：

1. 脑裂：虽然ZooKeeper能保证注册到节点上的所有监听器都会按顺序被触发，但并不能保证同一个时刻所有副本看到的状态是一样的，可能造成不同副本的响应不一致
2. 羊群效应：如果宕机的那个Broker的Partition数量很多，会造成多个Watch被触发，引起集群内大量的调整
3. 每个副本都要在ZK的Partition上注册Watcher，当集群内Partition数量很多时，会造成ZooKeeper负载过重

v2版本不再为replicas注册Replica变化的事件，而是放到broker级别的状态变化事件。而且v2的startReplica和v1的replicaStateChange功能是相似的，都是完成replicas发生变化时判断是否需要选举或成为follower。

![](/images/posts/kafka/v2_replica.png)

所有的brokers只对状态变化事件进行响应，而且状态变化事件也只由Leader决定，状态变化是通过onPartitionsReassigned写
到zk的/brokers/state/[broker_id]的请求事件队列触发的，而这个事件只注册在Partition的Leader Replica上。也
就是说follower状态的改变只基于partition的Leader发送请求时才会发生，如果leader没有对某个follower说要改变状态，则
follower是不会有任何事件发生的，这种方式类似于事件驱动的模式（引入了一个中间的状态机）。而第一版中任何一个replica改变，
都有可能导致所有其他follower都要做出相应（直接触发）。



### Controller要解决的问题

Replication的v3版本采用Controller实现，相比v2版本的主要不同点是：

1. Leader的变化从监听器改为由Controller管理
2. 控制器负责检测Broker的失败，并为每个受影响的Partition选举新的Leader
3. 控制器会将每个Leader的变化事件发送给受影响的每个Broker
4. 控制器和Broker之间的通信采用直接的RPC，而不是通过ZK队列

虽然因为引入了Controller，需要实现Controller的failover。但它的优点有：

从整个集群的所有Brokers中选举出一个Controller，它主要负责：

1. Partition的Leader变化事件
2. 新创建或删除一个topic
3. 重新分配Partition
4. 管理分区的状态机和副本的状态机

当控制器完成决策之后（决定了Partition新的Leader和ISR），它会将这个决策持久化到ZK中（将LeaderAndISR写入到ZK节点），并且
向所有受到影响的Brokers通过直接RPC的形式直接发送新的决策（发送LeaderAndISRRequest）。这些决策（持久化到ZK中的数据）是真理之源，
它们会被客户端用来路由请求（客户端只跟决策信息里的Leader交互），并且每个Broker启动的时候会用来恢复它的状态（Partition分配到Broker上，
说明Broker现在拥有了分配到的Partition）。当Broker启动之后，它会接收控制器通过直接RPC的形式发送过来的最新的决策。

每个KafkaServer中都会创建一个KafkaController对象，但是集群中只允许存在一个Leader Controller，这是通过ZooKeeper选举出来的：
第一个成功创建zk节点的那个Controller会成为Leader，其他的Controller会一直存在，但是并不会发挥作用。只有当原先的Controller挂掉后，
才会选举出新的Controller。集群中所有Brokers选举一个Controller和Partition中所有Replicas选举一个Leader是类似的。有点类
似于Hadoop-1.x中的SecondaryNameNode：同时启动NameNode和SecondaryNameNode，当NameNode挂掉后，SecondaryNameNode会代替
成为NameNode角色。在系统运行过程中，SecondaryNameNode要同步NameNode的数据，这样在NameNode发生故障时，SecondaryNameNode切
换为NameNode时数据并不会丢失或落后太多。Hadoop-2.x的HA使用了ZKFailoverController有两个Master：Active和Standby，工作原理和SNN
是类似的，只不过Active Master将信息写入共享存储，Standby从共享存储中读取信息，保持与Active Master的同步，从而减少故障时的切换
时间。不过KafkaController的failover机制并不会在系统运行过程中将其他所有Controller实例的数据和Leader同步，而是在Leader发生故障
时重新选举，并且重新恢复数据。

原先在没有Controller时，Leader或者Replica的变化都是通过监听器完成的，现在引入Controller之后，不仅要处理Broker的failover，也
要处理Controller的failover。Broker和Controller的failover也是通过注册在ZK上的Watcher监听器完成的，所有的Brokers监听一
个Controller，Controller也监听所有的Controller，两者互相关注。

**ZKQueue vs 直接RPC**

通过ZK完成Controller和brokers的通信是非常低效的，因为每次通信都需要2次ZK写（每个写操作花费2个RPC），
一次监听器的触发（写之后数据改变触发监听器调用），一次ZK读（监听器要获取最新的写操作结果），所以对于一次通信
就花费了6次RPC（2*2+1+1=6，RPC指的是在不同节点之间的通信，controller和brokers是在不同节点上，需要通过RPC通信才能读写数据）。

如果controller和所有的brokers之间为了能够直接通信，在Broker端实现一个admin RPC，那么每次通信只需要一个RPC。使用RPC意味
着当一个broker当掉后，它可能会丢失一些来自Controller的命令，所以在broker启动时它需要读取zk中保存的状态信息来恢复自己的状态。









