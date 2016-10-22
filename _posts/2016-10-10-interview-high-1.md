## Kafka

### why kafka so fast

Kafka is fast for a number of reasons. To name a few.

**Zero Copy** - See https://en.wikipedia.org/wiki/Zero-copy basically it calls the OS kernel direct rather
than at the application layer to move data fast.

**"Zero-copy"** describes computer operations in which the CPU does not perform the
task of copying data from one memory area to another. This is frequently used to
save CPU cycles and memory bandwidth when transmitting a file over a network.

As an example, reading a file and then sending it over a network the traditional way
requires two data copies and two context switches per read/write cycle. One of those data
copies use the CPU. Sending the same file via zero copy reduces the context switches to two
and eliminates all CPU data copies.[1]

Zero-copy protocols are especially important for high-speed networks in which the capacity of
a network link approaches or exceeds the CPU's processing capacity. In such a case the CPU spends
nearly all of its time copying transferred data, and thus becomes a bottleneck which limits the
communication rate to below the link's capacity. A rule of thumb used in the industry is that
roughly one CPU clock cycle is needed to process one bit of incoming data.

The Linux kernel supports zero-copy through various system calls, such as sys/socket.h's
sendfile, sendfile64, and splice.

Java input streams can support zero-copy through the java.nio.channels.FileChannel's
transferTo() method if the underlying operating system also supports zero copy.

RDMA (Remote Direct Memory Access) protocols deeply rely on zero-copy techniques.


**Batch Data in Chunks** - Kafka is all about batching the data into chunks. This minimises
cross machine latency with all the buffering/copying that accompanies this.

具体来讲: Producer 发送数据的时候, 数据会发送到

**Avoids Random Disk Access** - as Kafka is an immutable commit log it does not need to
rewind the disk and do many random I/O operations and can just access the disk in a
sequential manner. This enables it to get similar speeds from a physical disk compared with memory.

**Can Scale Horizontally** - The ability to have thousands of partitions for a single topic
spread among thousands of machines means Kafka can handle huge loads.


### Elasticsearch Near real time



### 系统设计 Scalability

这里就针对Scalability，有一些常见的优化技术，我就把他们列出 7脉神剑。

1. Cache: 缓存，万金油，哪里不行优先考虑
2. Queue: 消息队列，常见使用 Linkedin 的 kafka
3. Asynchronized: 批处理 + 异步, 减少系统 IO 瓶颈
4. Load Balance: 负载均衡，可以使用一致性 hash 技术做到尽量少的数据迁移
5. Parallelization: 并行计算，比如 MapReduce
6. Replication: 提高可靠性，如 HDFS，基于位置感知的多块拷贝
7. Partition: 数据库sharding, 通过 hash 取摸



### 进程和线程的区别

进程是操作系统分配资源的单位。在Windows下，进程又被细化为线程，也就是一个进程下有多个能独立运行的更小的单位。线程(Thread)是进程的一个实体，是CPU调度和分派的基本单位。线程不能够独立执行，必须依存在应用程序中，由应用程序提供多个线程执行控制。

线程和进程的关系是：线程是属于进程的，线程运行在进程空间内，同一进程所产生的线程共享同一内存空间，当进程退出时该进程所产生的线程都会被强制退出并清除。线程可与属于同一进程的其它线程共享进程所拥有的全部资源，但是其本身基本上不拥有系统资源，只拥有一点在运行中必不可少的信息(如程序计数器、一组寄存器和栈)。

在同一个时间里，同一个计算机系统中如果允许两个或两个以上的进程处于运行状态，这便是多任务。现代的操作系统几乎都是多任务操作系统，能够同时管理多个进程的运行。 多任务带来的好处是明显的，比如你可以边听mp3边上网，与此同时甚至可以将下载的文档打印出来，而这些任务之间丝毫不会相互干扰。那么这里就涉及到并行的问题，俗话说，一心不能二用，这对计算机也一样，原则上一个CPU只能分配给一个进程，以便运行这个进程。我们通常使用的计算机中只有一个CPU，也就是说只有一颗心，要让它一心多用，同时运行多个进程，就必须使用并发技术。实现并发技术相当复杂，最容易理解的是“时间片轮转进程调度算法”，它的思想简单介绍如下：在操作系统的管理下，所有正在运行的进程轮流使用CPU，每个进程允许占用CPU的时间非常短(比如10毫秒)，这样用户根本感觉不出来CPU是在轮流为多个进程服务，就好象所有的进程都在不间断地运行一样。但实际上在任何一个时间内有且仅有一个进程占有CPU。

引入线程带来的主要好处：

1. 在进程内创建、终止线程比创建、终止进程要快；
2. 同一进程内的线程间切换比进程间的切换要快,尤其是用户级线程间的切换。




### MPSC Lock Free Intrusive Linked Queue with state

[link](http://www.codeproject.com/Articles/870527/MPSC-Lock-Free-Intrusive-Linked-Queue-with-state)
[link](http://codereview.stackexchange.com/questions/224/thread-safe-and-lock-free-queue-implementation)

```java
public class MpscIntrusiveLinkedQueue<T>
{
    private final AtomicReferenceFieldUpdater<T, T> m_itemUpdater;
    private T m_head;
    private final AtomicReference<T> m_tail;

    public MpscIntrusiveLinkedQueue( AtomicReferenceFieldUpdater<T, T> itemUpdater ) {
        m_itemUpdater = itemUpdater;
        m_tail = new AtomicReference<T>();
    }

    public final boolean put( T item ) {
        assert( m_itemUpdater.get(item) == null );
        for (;;) {
            final T tail = m_tail.get();
            if (m_tail.compareAndSet(tail, item)) {
                if (tail == null) {
                    m_head = item;
                    return true;
                }
                else {
                    m_itemUpdater.set( tail, item );
                    return false;
                }
            }
        }
    }

    final T get() {
        assert( m_head != null );
        return m_head;
    }

    final T get_next() {
        assert( m_head != null );
        final T head = m_head;
        T next = m_itemUpdater.get( head );
        if (next == null) {
            m_head = null;
            if (m_tail.compareAndSet(head, null))
                return null;
            while ((next = m_itemUpdater.get(head)) == null);
        }
        m_itemUpdater.lazySet( head, null );
        m_head = next;
        return m_head;
    }
}
```
