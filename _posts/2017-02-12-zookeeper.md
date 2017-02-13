
## 基础

节点类型

1. EPHEMERAL
2. REGULAR
3. SEQUENCE


## 应用

### 一、数据发布/订阅(配置中心) - Publish/Subscribe

发布/订阅模式不仅仅会出现在这里，通常消息队列的设计中会更加常用到这种模式。发布/订阅模式功能可以分为这几种模式：

(1). Push模式是服务器主动将数据更新发送给所有订阅的客户端，优点是实时性好，但是服务端的工作比较繁重，常常需要记录各个客户端的状态信息，甚至要考虑消费者的消费能力实现流量控制工作；

(2). Pull模式则是由客户端主动发起请求来获取最新数据，通常采用定时机制轮训拉取的方式进行，所以实时性较差，但好处是实现简单；

(3). 还有就是结合两者进行推拉相结合的方式，客户端向服务端注册自己关心的主题，一旦主题数据有所更新，服务端会向对应订阅的客户端发送事件通知，客户端接收到事件通知后主动向服务器拉取最新的数据。

在系统开发中通常会遇到机器列表信息、运行时开关配置、数据库配置等信息，这些数据的特点是数据规模比较小、数据内容在运行时候常常发生动态变更、相同的数据在多个机器中共享。在单机的情况下，可以通过共享内存，或者统一配置文件(可以使用文件系统的特性监测文件变更iNotify)来满足这个需求，但是如果配置需要跨主机，尤其机器规模大且成员会发生变动的情况下共享这些动态信息就较为困难。

基于ZooKeeper的配置中心方案可以这样设计：首先运行一个ZooKeeper服务器集群，然后在其上面创建周知路径的 ZNode; 专用的配置管理程序可以修改 ZNode 上的配置信息，作为用户更新配置的操作接口；关心这些配置的分布式应用启动时候主动获取配置信息一次，然后对 ZNode 注册 Watcher 监听，那么当 ZNode 的数据内容发生变化后，ZooKeeper 就可以将这个变更通知发送给所有的客户端，客户端得知这个变更通知后就可以请求获取最新数据。通过这种手段，就可以实现配置信息的集中式管理和动态更新部署的功能了。

## 二、负载均衡 - Load Balance

负载均衡在前面的Nginx中也介绍过了，Nginx支持3层和7层软件负载均衡，对用户看来这实际上是实现的基于服务端的负载均衡操作。其实负载均衡还可以在客户端实现，之前介绍的memcached的负载均衡基本都是在客户端通过一致性hash原理实现的。(也就是前向和后向代理)

通过ZooKeeper实现的负载均衡也是通过客户端来实现的：ZooKeeper创建一个周知路径的ZNode，其数据内容包含了可以提供服务的服务器地址信息，接收服务的客户端在该ZNode上注册Watcher以侦听它的改变。在工作的时候客户端获取提供服务的服务器列表，在接收到修改事前之前可以缓存该列表提高性能，然后服务调用者可以采用自己某种特定负载均衡算法(比如Round Robin、HASH、响应时间、最少连接等算法)选取机器获取服务。

为了使用方便，服务机器列表的配置可以采用全自动的方式，这也涉及到机器健康度检测问题。可以设计一个健康度探测服务端，并负责更新ZNode中的机器列表：健康度探测服务端可以同机器中的列表建立TCP长连接，健康度探测服务端采用Ping-Pong心跳监测的方式进行健康度监测；也可以机器每隔一定的时间向这个健康度探测服务端发送数据包，如果健康度探测在某个超时时间后仍未收到某机器的数据包，就认定其不可用而将其从机器列表中进行删除。(呃，后面想想，每个服务端机器把自己作为临时ZNode创建在某个路径下，这样的侦测操作不是更加的方便？和集群管理重了)

## 命名服务 - Name Service

命名服务其实是指明了一种映射关系，在分布式开发中有很多的资源需要命名，比如主机地址，RPC服务列表等，客户端根据名字可以获取资源的实体、服务地址和提供者等信息。

ZooKeeper提供了方便的API，可以轻松的创建全局唯一的path，这个path就可以作为名称使用，所以ZooKeeper的Name Service是开箱即用的。而且ZNode支持SEQUENTIAL属性，通过创建顺序节点的手法就可以创建具有相同名字但带顺序后缀的这样很具有规则性的名字，这样的命名服务显然在保证命名唯一性的同时更具有解释意义。

提供命名服务的还有 netflix 的 eruka

## 四、分布式协调/通知 - Coordinator

得益于ZooKeeper特有的Watcher注册和异步通知的机制，可以实现分布式环境下不同机器，甚至是不同系统之间的通知和协调工作，以应对于数据变更的实时快速处理，这是和观察者类似的工作模式。而且通过ZooKeepr作为一个事件的中介者，也起到了业务之间解除耦合的效果。

## 五、集群管理 - Management

用于对集群中的机器进行监控的情况，主要包括对集群运行状态的收集，以及对集群和集群成员进行操作和控制。

传统的运维都是基于Agent的分布式集群管理模式，通过在集群成员上部署该类服务软件，该软件负责主动向集群控制中心汇报机器的状态信息。这种方式虽然直接，但是具有着固有的缺陷：很难实现跟业务相关的细化监控，通常都是对CPU、负载等通用信息进行监测；如果整个集群环境一致还可以，否则就必须面对集群中的异构成员进行兼容和适配的问题。

如果运行一个ZooKeeper集群，不仅通过临时节点在会话结束后会自动消失的特性，可以快速侦测机器的上下线，而且可以通过创建特定的ZNode，集群中的机器就可以向控制中心报告更多主机相关甚至业务相关的信息，而且这一切操作都极为便利，只需要在业务端和控制中心进行适配就可以了。

## 六、Master选举 - Election

可以应用于那些只需要一个主节点做特殊的工作，其他节点做普通的工作或者候命作为冗余节点的情况，比如数据库的读写分离可以让Master处理所有的写任务，而剩余的读请求由其他机器负载均衡完成，借此提供整个数据库性能；类似的Master可以单独处理一些复杂的逻辑或者计算，而将计算结果同步共享给集群中的其他主机使用，以此减少重复劳动提高资源利用率。

传统情况下的选主可以使用 **数据库唯一性索引** 简单实现，在此约束下所有的机器创建同一条记录时候只能有一个机器成功，那么可以让这个独一无二的机器作为Master，但是这种方法只是解决了竞争冲突问题，无法解决Master单点故障后整个系统不可用的问题，即不能实现高效快速的动态Master选举功能。

对此，ZooKeeper的强一致性可以天然解决Master选举的问题(注意这里的选主是客户端的Master，和ZAB协议中的Leader没有关系)：首先ZooKeeper保证在多个客户端请求创建同一路径描述的ZNode的情况下，只会有一个客户端的请求成功；其次创建ZNode可以是临时ZNode，那么一旦创建这个临时ZNode的Master挂掉后会导致会话结束，这个临时ZNode就会自动消失；在之前竞争Master失败的客户端，可以注册该ZNode的Watcher侦听，一旦接收到节点的变更事件，就表示Master不可用了，此时大家就可以即刻再次发起Master选举操作了，以实现了一种高可用的automatic fail-over机制，满足了机器在线率有较高要求的应用场景。

除了上面的方式，各个主机还可以通过创建临时顺序ZNode的方式，每个主机会具有不同的后缀，一旦当前的Master宕机之后自动轮训下一个可用机器，而下线的机器也可以随时再次上线创建新序列号的临时顺序节点。

A simple way of doing leader election with ZooKeeper is to use the SEQUENCE|EPHEMERAL flags when creating znodes that represent "proposals" of clients. The idea is to have a znode, say "/election", such that each znode creates a child znode "/election/guid-n_" with both flags SEQUENCE|EPHEMERAL. With the sequence flag, ZooKeeper automatically appends a sequence number that is greater than any one previously appended to a child of "/election". The process that created the znode with the smallest appended sequence number is the leader.

That's not all, though. It is important to watch for failures of the leader, so that a new client arises as the new leader in the case the current leader fails. A trivial solution is to have all application processes watching upon the current smallest znode, and checking if they are the new leader when the smallest znode goes away (note that the smallest znode will go away if the leader fails because the node is ephemeral). But this causes a herd effect: upon a failure of the current leader, all other processes receive a notification, and execute getChildren on "/election" to obtain the current list of children of "/election". If the number of clients is large, it causes a spike on the number of operations that ZooKeeper servers have to process. To avoid the herd effect, it is sufficient to watch for the next znode down on the sequence of znodes. If a client receives a notification that the znode it is watching is gone, then it becomes the new leader in the case that there is no smaller znode. Note that this avoids the herd effect by not having all clients watching the same znode.

Let ELECTION be a path of choice of the application. To volunteer to be a leader:

Create znode z with path "ELECTION/guid-n_" with both SEQUENCE and EPHEMERAL flags;
Let C be the children of "ELECTION", and i be the sequence number of z;
Watch for changes on "ELECTION/guid-n_j", where j is the largest sequence number such that j < i and n_j is a znode in C;
Upon receiving a notification of znode deletion:

Let C be the new set of children of ELECTION
If z is the smallest node in C, then execute leader procedure;
Otherwise, watch for changes on "ELECTION/guid-n_j", where j is the largest sequence number such that j < i and n_j is a znode in C;

## 七、分布式锁 - Lock

主要用来进行分布式系统之间的访问资源的同步手段。在使用中分布式锁可以支持这些服务：保持独占、共享使用和时序访问。虽然关系数据库在更新的时候，数据库系统会根据隔离等级自动使用行锁、表锁机制保证数据的完整性，但是数据库通常都是大型系统的性能瓶颈之所在，所以如果使用分布式锁可以起到一定的协调作用，那么可以期待增加系统的运行效率。

保持独占，就是当多个客户端试图获取这把锁的时候，只能有一个可以成功获得该锁，跟上面的Master选举比较类似，当多个竞争者同时尝试创建某个path(例如”_locknode_/guid-lock-“)的ZNode时候，ZooKeeper的一致性能够保证只有一个客户端成功，创建成功的客户端也就拥有了这把互斥锁，此时其他客户端可以在这个ZNode上面注册Watcher侦听，以便得到锁变更(如持锁客户端宕机、主动解锁)的情况采取接下来的操作。

**共享使用** 类似于读写锁的功能，多个读锁可以同时对一个事务可见，但是读锁和写锁以及写锁和写锁之间是互斥的。锁的名字按照类似”/share_lock/[Host]-R/W-SN”的形式创建临时顺序节点，在创建锁的同时读取/share_lock节点下的所有子节点，并注册对/share_lock/节点的Watcher侦听，然后依据读写所的兼容法则检查比自己序号小的节点看是否可以满足当前操作请求，如果不满足就执行等待。当持有锁的节点崩溃或者释放锁之后，所有处于等待状态的节点就都得到了通知，实际中这会产生一个“惊群效应”，所以可以在上面注册/share_lock/的Watcher事件进行细化，只注册比自己小的那个子节点的Watcher侦听就可以了，以避免不必要的唤醒。

**时序访问** 的方式，则是在这个时候每个请求锁的客户端都可以创建临时顺序ZNode的子节点，他们维系着一个带有序列号的后缀，同时添加对锁节点(或者像上面类似优化，只注册序列号比自己小的那个子节点)的Watcher侦听，这样前面的客户端释放锁之后，后面的客户端会得到事件通知，然后按照一定顺序接下来的一个客户端获得锁。该模式能够保证每个客户端都具有访问的机会，但是其是按照创建临时顺序子节点的顺序按次序依次访问的。

## 八、分布式队列 - Queue

说到分布式队列，目前已有相当多的成熟消息中间件了。在ZooKeeper的基础上，可以方便地创建先进先出队列，以及对数据进行聚集之后再统一安排处理的Barrier的工作模式。

先入先出之FIFO队列算是最常见使用的数据模型了，类似于一个生产者-消费者的工作模型。生产者会创建顺序ZNode，这些顺序ZNode的后缀表明了创建的顺序，消费者获得这些顺序ZNode后挑出序列号最小的进行消费，就简单的实现了FIFO的数据类型。

对于另外一种Barrier(叫做屏障)工作模式，是需要把数据集聚之后再做统一处理，通常在大规模并行计算的场景上会使用到。这种队列实现也很简单，就是在创建顺序ZNode的时候记录队列当前已经拥有的事务数目，如果达到了Barrier的数目，就表示条件就绪了于是创建类似“/synchronizing/start”的ZNode，而等待处理的消费者之前就侦听该ZNode是否被创建，如果侦测到一旦创建了就表明事务的数目满足需求了，于是可以启动处理工作。

## Barrier

Distributed systems use barriers to block processing of a set of nodes until a condition is met at which time all the nodes are allowed to proceed. Barriers are implemented in ZooKeeper by designating a barrier node. The barrier is in place if the barrier node exists. Here's the pseudo code:

1. Client calls the ZooKeeper API's exists() function on the barrier node, with watch set to true.
2. If exists() returns false, the barrier is gone and the client proceeds
3. Else, if exists() returns true, the clients wait for a watch event from ZooKeeper for the barrier node.
4. When the watch event is triggered, the client reissues the exists( ) call, again waiting until the barrier node is removed.


**Double Barriers**

Double barriers enable clients to synchronize the beginning and the end of a computation. When enough processes have joined the barrier, processes start their computation and leave the barrier once they have finished. This recipe shows how to use a ZooKeeper node as a barrier.

The pseudo code in this recipe represents the barrier node as b. Every client process p registers with the barrier node on entry and unregisters when it is ready to leave. A node registers with the barrier node via the Enter procedure below, it waits until x client process register before proceeding with the computation. (The x here is up to you to determine for your system.)

**Enter**

1. Create a name n = b+“/”+p
2. Set watch: exists(b + ‘‘/ready’’, true)
3. Create child: create( n, EPHEMERAL)
4. L = getChildren(b, false)
5. if fewer children in L than x, wait for watch event
6. else create(b + ‘‘/ready’’, REGULAR)

**Leave**

1. L = getChildren(b, false)
2. if no children, exit
3. if p is only process node in L, delete(n) and exit
4. if p is the lowest process node in L, wait on highest process node in L
5. else delete(n) if still exists and wait on lowest process node in L
6. goto 1

On entering, all processes watch on a ready node and create an ephemeral node as a child of the barrier node. Each process but the last enters the barrier and waits for the ready node to appear at line 5. The process that creates the xth node, the last process, will see x nodes in the list of children and create the ready node, waking up the other processes. Note that waiting processes wake up only when it is time to exit, so waiting is efficient.

On exit, you can't use a flag such as ready because you are watching for process nodes to go away. By using ephemeral nodes, processes that fail after the barrier has been entered do not prevent correct processes from finishing. When processes are ready to leave, they need to delete their process nodes and wait for all other processes to do the same.

Processes exit when there are no process nodes left as children of b. However, as an efficiency, you can use the lowest process node as the ready flag. All other processes that are ready to exit watch for the lowest existing process node to go away, and the owner of the lowest process watches for any other process node (picking the highest for simplicity) to go away. This means that only a single process wakes up on each node deletion except for the last node, which wakes up everyone when it is removed.



## Usage

## reference
[1] http://zookeeper.apache.org/doc/trunk/recipes.html
