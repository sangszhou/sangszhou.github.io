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























