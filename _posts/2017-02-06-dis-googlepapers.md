---
layout: post
title: MapReduce GFS Bigtable
categories: [dis]
keywords: MapReduce, GFS, Bigtable
---

## MapReduce

**FastPath for mapreduce**

1. 在执行任务以前，框架会把输入的文件拆分成 16 ~ 64M 大小的文件
2. map 函数会生成 R 个文件，对应 R 个 reducer. combine 是 local reducer，它的类型和 reducer 完全一致，可以在 main 函数中声明 combiner
3. master 会管理整个集群信息，如果 map 失败，则重做 map, 只有当 map 的结果全部结束，才算完
4. master 会记录 task 的状态，inProgress, finish 等等
5. master 宕机以后，整个任务全部终止，接下来如何进行要看 client 端的配置
6. 如果有部分 task 比较慢，则 master 会创建 backup, 正确快速做完
7. 硬件的假设和GFS相同

MapReduce是将一个大作业拆分为多个小作业的框架（大作业和小作业应该本质是一样的，只是规模不同），用户需要做的就是决定拆成多少份，以及定义作业本身

![](/images/posts/bigdata/mapreduce.png)

**map函数和reduce函数**

上图是论文里给出的流程图。一切都是从最上方的user program开始的，user program链接了MapReduce库，实现了最基本的Map函数和Reduce函数。图中执行的顺序都用数字标记了。

1. MapReduce库先把user program的输入文件划分为M份（M为用户定义），每一份通常有16MB到64MB，如图左方所示分成了split0~4；然后使用fork将用户进程拷贝到集群内其它机器上。
2. user program的副本中有一个称为master，其余称为worker，master是负责调度的，为空闲worker分配作业（Map作业或者Reduce作业），worker的数量也是可以由用户指定的。
3. 被分配了Map作业的worker，开始读取对应分片的输入数据，**Map作业数量是由M决定的**，和split一一对应；Map作业从输入数据中抽取出键值对，每一个键值对都作为参数传递给map函数，map函数产生的中间键值对被缓存在内存中。
4. 缓存的中间键值对会被定期写入本地磁盘，而且被分为R个区，R的大小是由用户定义的，将来每个区会对应一个Reduce作业；这些中间键值对的位置会被通报给master，master负责将信息转发给Reduce worker。
5. master通知分配了Reduce作业的worker它负责的分区在什么位置（肯定不止一个地方，每个Map作业产生的中间键值对都可能映射到所有R个不同分区），当Reduce worker把所有它负责的中间键值对都读过来后，先对它们进行排序，使得相同键的键值对聚集在一起。因为不同的键可能会映射到同一个分区也就是同一个Reduce作业（谁让分区少呢），所以排序是必须的。
6. reduce worker遍历排序后的中间键值对，对于每个唯一的键，都将键与关联的值传递给reduce函数，reduce函数产生的输出会添加到这个分区的输出文件中。
7. 当所有的Map和Reduce作业都完成了，master唤醒正版的user program，MapReduce函数调用返回user program的代码。

所有执行完毕后，MapReduce 输出放在了 R 个分区的输出文件中（分别对应一个Reduce作业）。用户通常并不需要合并这R个文件，而是将其作为输入交给另一个MapReduce程序处理。整个过程中，输入数据是来自底层分布式文件系统（GFS）的，中间数据是放在本地文件系统的，最终输出数据是写入底层分布式文件系统（GFS）的。而且我们要注意Map/Reduce作业和map/reduce函数的区别：Map作业处理一个输入数据的分片，可能需要调用多次map函数来处理每个输入键值对；Reduce作业处理一个分区的中间键值对，期间要对每个不同的键调用一次reduce函数，Reduce作业最终也对应一个输出文件。

我更喜欢把流程分为三个阶段。第一阶段是准备阶段，包括1、2，主角是MapReduce库，完成拆分作业和拷贝用户程序等任务；第二阶段是运行阶段，包括3、4、5、6，主角是用户定义的map和reduce函数，每个小作业都独立运行着；第三阶段是扫尾阶段，这时作业已经完成，作业结果被放在输出文件里，就看用户想怎么处理这些输出了。




## GFS

GFS的主要假设如下：

1. GFS的服务器都是普通的商用计算机，并不那么可靠，集群出现结点故障是常态。因此必须时刻监控系统的结点状态，当结点失效时，必须能检测到，并恢复之。
2. 系统存储适当数量的大文件。理想的负载是几百万个文件，文件一般都超过100MB，GB级别以上的文件是很常见的，必须进行有效管理。支持小文件，但不对其进行优化。
3. 负载通常包含两种读：大型的流式读（顺序读），和小型的随机读。前者通常一次读数百KB以上，后者通常在随机位置读几个KB。
4. 负载还包括很多连续的写操作，往文件追加数据（append）。文件很少会被修改，支持随机写操作，但不必进行优化。
5. 系统必须实现良好定义的语义，用于多客户端并发写同一个文件。同步的开销必须保证最小。
6. 高带宽比低延迟更重要，GFS的应用大多需要快速处理大量的数据，很少会严格要求单一操作的响应时间。

### 体系结构


![](/images/posts/bigdata/gfs_architecture.jpg)

GFS包括一个master结点（元数据服务器），多个chunkserver（数据服务器）和多个client（运行各种应用的客户端）。

chunkserver提供存储。GFS会将文件划分为定长数据块，每个数据块都有一个全局唯一不可变的id（chunk_handle），数据块以普通Linux文件的形式存储在chunkserver上，出于可靠性考虑，每个数据块会存储多个副本，分布在不同chunkserver。

GFS master就是GFS的元数据服务器，负责维护文件系统的元数据，包括命名空间、访问控制、文件-块映射、块地址等，以及控制系统级活动，如垃圾回收、负载均衡等。

应用需要链接client的代码，然后client作为代理与master和chunkserver交互。master会定期与chunkserver交流（心跳），以获取chunkserver的状态并发送指令。

GFS为数据块设置了一个很大的长度，64MB，这比传统文件系统的块长要大多了。大块长会带来很多好处：

1. 减少client和master的交互次数，因为读写同一个块只需要一次交互，在GFS假设的顺序读写负载的场景下特别有用；
2. 同样也减少了client和chunkserver的交互次数，降低TCP/IP连接等网络开销；
3. 减少了元数据的规模，因此master可以将元数据完全放在内存，这对于集中式元数据模型的GFS尤为重要。

大数据块也有缺点。最大的缺点可能就是内部碎片了，不过考虑到文件一般都相当大，所以碎片也只存在于文件的最后一个数据块。还有一个缺点不是那么容易看出来，由于小文件可能只有少量数据块，极端情况只有一个，那么当这个小文件是热点文件时，存储该文件数据块的chunkserver可能会负载过重。不过正如前面所说，小文件不在GFS的优化范围。

为了提高数据的可靠性和并发性，每一个数据块都有多个副本。当客户端请求一个数据块时，master会将所有副本的地址都通知客户端，客户端再择优（距离最短等）选择一个副本。

### 元数据服务

GFS是典型的集中式元数据服务，所有的元数据都存放在一个master结点内。元数据主要包括三种：**文件和数据块的命名空间**，**文件-数据块映射表**，**数据块的副本位置**。所有的元数据都是放在内存里的。

前两种元数据会被持久化到本地磁盘中，以操作日志的形式。操作日志会记录下这两种元数据的每一次关键变化，因此当master宕机，就可以根据日志恢复到某个时间点。日志的意义还在于，它提供了一个时间线，用于定义操作的先后顺序，文件、数据块的版本都依赖于这个时间顺序。

数据块的副本位置则没有持久化，因为动辄数以百计的chunkserver是很容易出错的，因此只有chunkserver对自己存储的数据块有绝对的话语权，而master上的位置信息很容易因为结点失效等原因而过时。取而代之的方法是，master启动时询问每个chunkserver的数据块情况，而且chunkserver在定期的心跳检查中也会汇报自己存储的部分数据块情况。

### 缓存和预取

GFS的客户端和chunkserver都不会缓存任何数据，这是因为GFS的典型应用是顺序访问大文件，不存在时间局部性。空间局部性虽然存在，但是数据集一般很大，以致没有足够的空间缓存。

我们知道集中式元数据模型的元数据服务器容易成为瓶颈，应该尽量减少客户端与元数据服务器的交互。因此GFS设计了元数据缓存。client需要访问数据时，先询问master数据在哪儿，然后将这个数据地址信息缓存起来，之后client对该数据块的操作都只需直接与chunkserver联系了，当然缓存的时间是有限的，过期作废。

master还会元数据预取。因为空间局部性是存在，master可以将逻辑上连续的几个数据块的地址信息一并发给客户端，客户端缓存这些元数据，以后需要时就可以不用找master的麻烦了。

### 完整性

GFS使用数以千计的磁盘，磁盘出错导致数据被破坏时有发生，我们可以用其它副本来恢复数据，但首先必须能检测出错误。chunksever会使用校验和来检测错误数据。每一个块（chunk）都被划分为64KB的单元（block），每个block对应一个32位的校验和。校验和与数据分开存储，内存有一份，然后以日志的形式在磁盘备一份。

chunkserver在发送数据之前会核对数据的校验和，防止错误的数据传播出去。如果校验和与数据不匹配，就返回错误，并且向master反映情况。master会开始克隆副本的操作，完成后就命令该chunkserver删除非法副本。


### 一致性

一致性指的是master的元数据和chunkserver的数据是否一致，多个数据块副本之间是否一致，多个客户端看到的数据是否一致。

![图2](/images/posts/bigdata/gfs_write_flow.jpg)

写失败了是否会回滚，还是不做任何后续处理？

图2是写操作的控制流和数据流：

1. client需要更新一个数据块，询问master谁拥有该数据块的租约（谁是primary）；
2. master将持有租约的primary和其它副本的位置告知client，client缓存之；
3. client向所有副本传输数据，这里副本没有先后顺序，根据网络拓扑情况找出最短路径，数据从client出发沿着路径流向各个chunkserver，这个过程采用流水线（网络和存储并行）。chunkserver将数据放到LRU缓存；
4. 一旦所有的副本都确定接受数据，client向primary发送写请求，primary为这个前面接受到的数据分配序列号（primary为所有的写操作分配连续的序列号表示先后顺序），并且按照顺序执行数据更新；
5. primary将写请求发送给其它副本，每个副本都按照primary确定的顺序执行更新；
6. 其它副本向primary汇报操作情况；
7. primary回复client操作情况，任何副本错误都导致此次请求失败，并且此时副本处于不一致状态（写操作完成情况不一样）。client会尝试几次3到7的步骤，实在不行就只能重头来过了。

如果一个写请求太大了或者跨越了chunk，GFS的client会将其拆分为多个写请求，每个写请求都遵循上述过程，但是可能和其它应用的写操作交叉在一起。所以这些写操作共享的数据区域就可能包含几个写请求的碎片（就是下文提到的undefined状态）。

GFS还提供另一种写操作**append record**，append只在文件的尾部以record为单位（为了避免内部碎片，record一般不会很大）写入数据。append是原子性的，GFS保证将数据顺序地写到文件尾部至少一次。append record的流程和图2类似，只是在primary有点区别，GFS必须保证一个record存储在一个chunk内，所以当primary判断当前chunk无足够空间时，就通知所有副本将chunk填充，然后汇报失败，client会申请创建新chunk并重新执行一次append record操作。如果该chunk大小合适，primary会将数据写到数据块的尾部，然后通知其它副本将数据写到一样的偏移。任何副本append失败，各个副本会处于不一致状态（完成或未完成），这时primary必然是成功写入的（不然就没有4以后的操作了）。客户端重试append record操作时，因为primary的chunk长度已经变化了，primary就必须在新的偏移写入数据，而其它副本也是照做。这就导致上一个失败的偏移位置，各个副本处于不一致状态，应用必须自己区分record正确与否，我称这为无效数据区。

表1说明了GFS的一致性保证，明白write和append操作后就容易理解了。consistent指的是多个副本之间是完全一致的；defined指的是副本数据修改后不仅一致而且client能看到其修改的数据全貌。

1. 成功的连续write是已定义的，各个副本数据一致；
2. 成功的并发write能保证一致性性，即各个副本是一样的，但数据并不一定如用户所期望，如前所述，多个用户的修改可能交错在一起；
3. 失败的write操作，使得副本之间不一致，而且数据undefined，不同client可能看到不同的数据（注意区别defined、undefined数据的方法）；
4. 成功的append操作，不管是顺序还是并发都是defined，因为GFS保证了append是原子性的（atomically at least once）。有效数据区确实是defined的，但是失败append操作留下的无效数据区可能会有不一致的情况，所以中间可能零散分布着不一致的数据。

如上所述，在primary的协调下，能保证并发write的一致性。但还有一些可能会导致数据不一致，比如chunkserver宕机错过了数据更新，这时就会出现新旧版本的数据，GFS为每个数据块分配版本来解决这个问题。master每次授权数据块租约给primary之前，都会增加数据块的版本号，并且通知所有副本更新版本号。客户端需要读数据时当然会根据这个版本号来判断副本的数据是否最新。

### 可用性

为了保证数据的可用性，GFS为每个数据块存储了多个副本，在数据的布局里有介绍，这里主要关注下元数据的可用性。

GFS是典型的集中式元数据模型，一个元数据服务器承担了巨大的压力，不仅有性能的问题，还有单点故障问题。master为了能快速从故障中恢复过来，采用了log和checkpoint技术，log记录了元数据的每一次变化。用咱们备份的话来说，checkpoint就相当于一次全量备份，log相当于连续数据保护，master宕机后，就先恢复到checkpoint，然后根据log恢复到最新状态。每次创建一个新的checkpoint，log就可以清空，这有效控制了log的大小。

这还不够，如果master完全坏了肿么办？GFS设置了“影子”服务器，master将日志备份到影子上，影子按照日志把元数据与master同步。如果master悲剧了，我们还有影子。平时正常工作时，影子可以分担一部分元数据访问工作，当然不提供直接的写操作。



## BigTable

Bigtable是一个为管理大规模半结构化数据而设计的分布式存储系统，可以扩展到PB级数据和上千台服务器。很多google的项目使用Bigtable存储数据，这些应用对Bigtable提出了不同的挑战，比如数据规模的要求、延迟的要求。Bigtable能满足这些多变的要求，为这些产品成功地提供了灵活、高性能的存储解决方案。

Bigtable看起来像一个数据库，采用了很多数据库的实现策略。但是Bigtable并不支持完整的关系型数据模型；而是为客户端提供了一种简单的数据模型，客户端可以动态地控制数据的布局和格式，并且利用底层数据存储的局部性特征。Bigtable将数据统统看成无意义的字节串，客户端需要将结构化和非结构化数据串行化再存入Bigtable。

本质上说，Bigtable是一个键值（key-value）映射。按作者的说法，Bigtable是一个稀疏的，分布式的，持久化的，多维的排序映射。

先来看看多维、排序、映射。Bigtable的键有三维，分别是行键（row key）、列键（column key）和时间戳（timestamp），行键和列键都是字节串，时间戳是64位整型；而值是一个字节串。可以用 (row:string, column:string, time:int64)→string 来表示一条键值对记录。

行键可以是任意字节串，通常有10-100字节。行的读写都是原子性的。Bigtable按照行键的字典序存储数据。Bigtable的表会根据行键自动划分为片（tablet），片是负载均衡的单元。最初表都只有一个片，但随着表不断增大，片会自动分裂，片的大小控制在100-200MB。行是表的第一级索引，我们可以把该行的列、时间和值看成一个整体，简化为一维键值映射，类似于：


列是第二级索引，每行拥有的列是不受限制的，可以随时增加减少。为了方便管理，列被分为多个列族（column family，是访问控制的单元），一个列族里的列一般存储相同类型的数据。一行的列族很少变化，但是列族里的列可以随意添加删除。列键按照family:qualifier格式命名的。这次我们将列拿出来，将时间和值看成一个整体，简化为二维键值映射，类似于：


```
table{  
  // ...  
  "aaaaa" : { //一行  
    "A" : { //列族A  
      "foo" : {sth.}, //一列  
      "bar" : {sth.}  
    },  
    "B" : { //列族B  
      "" : {sth.}  
    }  
  },  
  "aaaab" : { //一行  
    "A" : {  
      "foo" : {sth.},  
    },  
    "B" : {  
      "" : "ocean"  
    }  
  },  
  // ...  
}  
```

时间戳是第三级索引。Bigtable允许保存数据的多个版本，版本区分的依据就是时间戳。时间戳可以由Bigtable赋值，代表数据进入Bigtable的准确时间，也可以由客户端赋值。数据的不同版本按照时间戳降序存储，因此先读到的是最新版本的数据。我们加入时间戳后，就得到了Bigtable的完整数据模型，类似于：

查询时，如果只给出行列，那么返回的是最新版本的数据；如果给出了行列时间戳，那么返回的是时间小于或等于时间戳的数据。比如，我们查询"aaaaa"/"A:foo"，返回的值是"y"；查询"aaaaa"/"A:foo"/10，返回的结果就是"m"；查询"aaaaa"/"A:foo"/2，返回的结果是空。

图1是Bigtable论文里给出的例子，Webtable表存储了大量的网页和相关信息。在Webtable，每一行存储一个网页，其反转的url作为行键，比如maps.google.com/index.html的数据存储在键为com.google.maps/index.html的行里，反转的原因是为了让同一个域名下的子域名网页能聚集在一起。图1中的列族"anchor"保存了该网页的引用站点（比如引用了CNN主页的站点），qualifier是引用站点的名称，而数据是链接文本；列族"contents"保存的是网页的内容，这个列族只有一个空列"contents:"。图1中"contents:"列下保存了网页的三个版本，我们可以用("com.cnn.www", "contents:", t5)来找到CNN主页在t5时刻的内容。

再来看看作者说的其它特征：稀疏，分布式，持久化。持久化的意思很简单，Bigtable的数据最终会以文件的形式放到GFS去。Bigtable建立在GFS之上本身就意味着分布式，当然分布式的意义还不仅限于此。稀疏的意思是，一个表里不同的行，列可能完完全全不一样。

### 支撑技术

Bigtable依赖于google的几项技术。用GFS来存储日志和数据文件；按SSTable文件格式存储数据；用Chubby管理元数据。

SSTable的全称是Sorted Strings Table，是一种不可修改的有序的键值映射，提供了查询、遍历等功能。每个SSTable由一系列的块（block）组成，Bigtable将块默认设为64KB。在SSTable的尾部存储着块索引，在访问SSTable时，整个索引会被读入内存。BigTable论文没有提到SSTable的具体结构，LevelDb日知录之四： SSTable文件这篇文章对LevelDb的SSTable格式进行了介绍，因为LevelDB的作者JeffreyDean正是BigTable的设计师，所以极具参考价值。每一个片（tablet）在GFS里都是按照SSTable的格式存储的，每个片可能对应多个SSTable。

Chubby是一种高可用的分布式锁服务，Chubby有五个活跃副本，同时只有一个主副本提供服务，副本之间用Paxos算法维持一致性，Chubby提供了一个命名空间（包括一些目录和文件），每个目录和文件就是一个锁，Chubby的客户端必须和Chubby保持会话，客户端的会话若过期则会丢失所有的锁。关于Chubby的详细信息可以看google的另一篇论文：The Chubby lock service for loosely-coupled distributed systems。Chubby用于片定位，片服务器的状态监控，访问控制列表存储等任务。

### Bigtable集群

FastPath:

和 GFS 的关系。我觉得最大的关系是，BigTable 不再管每个表的备份，表的多个备份完全由 GFS 保证，这样简化了 BigTable 很多工作。

Bigtable集群包括三个主要部分：一个供客户端使用的库，一个主服务器（master server），许多片服务器（tablet server）。

正如数据模型小节所说，Bigtable会将表（table）进行分片，片（tablet）的大小维持在100-200MB范围，一旦超出范围就将分裂成更小的片，或者合并成更大的片。每个片服务器负责一定量的片，处理对其片的读写请求，以及片的分裂或合并。片服务器可以根据负载随时添加和删除。这里片服务器并不真实存储数据，而相当于一个连接Bigtable和GFS的代理，客户端的一些数据操作都通过**片服务器**代理间接访问GFS。

主服务器负责将片分配给片服务器，监控片服务器的添加和删除，平衡片服务器的负载，处理表和列族的创建等。注意，主服务器不存储任何片，不提供任何数据服务，也不提供片的定位信息。

客户端需要读写数据时，直接与片服务器联系。因为客户端并不需要从主服务器获取片的位置信息，所以大多数客户端从来不需要访问主服务器，主服务器的负载一般很轻。

![](/images/posts/bigdata/bigtable-index.jpg)

**片的定位**

前面提到主服务器不提供片的位置信息，那么客户端是如何访问片的呢？来看看论文给的示意图，Bigtable使用一个类似B+树的数据结构存储片的位置信息。

首先是第一层，Chubby file。这一层是一个Chubby文件，它保存着root tablet的位置。这个Chubby文件属于Chubby服务的一部分，一旦Chubby不可用，就意味着丢失了root tablet的位置，整个Bigtable也就不可用了。

第二层是root tablet。root tablet其实是元数据表（METADATA table）的第一个分片，它保存着元数据表其它片的位置。root tablet很特别，为了保证树的深度不变，root tablet从不分裂。

第三层是其它的元数据片，它们和root tablet一起组成完整的元数据表。每个元数据片都包含了许多用户片的位置信息。

可以看出整个定位系统其实只是两部分，一个Chubby文件，一个元数据表。注意元数据表虽然特殊，但也仍然服从前文的数据模型，每个分片也都是由专门的片服务器负责，这就是不需要主服务器提供位置信息的原因。客户端会缓存片的位置信息，如果在缓存里找不到一个片的位置信息，就需要查找这个三层结构了，包括访问一次Chubby服务，访问两次片服务器。

**片的存储和访问**

片的数据最终还是写到GFS里的，片在GFS里的物理形态就是若干个SSTable文件。图5展示了读写操作基本情况。

当片服务器收到一个写请求，片服务器首先检查请求是否合法。如果合法，先将写请求提交到日志去，然后将数据写入内存中的memtable。memtable相当于SSTable的缓存，当memtable成长到一定规模会被冻结，Bigtable随之创建一个新的memtable，并且将冻结的memtable转换为SSTable格式写入GFS，这个操作称为minor compaction。

当片服务器收到一个读请求，同样要检查请求是否合法。如果合法，这个读操作会查看所有SSTable文件和memtable的合并视图，因为SSTable和memtable本身都是已排序的，所以合并相当快。

每一次minor compaction都会产生一个新的SSTable文件，SSTable文件太多读操作的效率就降低了，所以Bigtable定期执行merging compaction操作，将几个SSTable和memtable合并为一个新的SSTable。BigTable还有个更厉害的叫major compaction，它将所有SSTable合并为一个新的SSTable。

### BigTable和GFS的关系

集群包括主服务器和片服务器，主服务器负责将片分配给片服务器，而具体的数据服务则全权由片服务器负责。但是不要误以为片服务器真的存储了数据（除了内存中memtable的数据），数据的真实位置只有GFS才知道，主服务器将片分配给片服务器的意思应该是，片服务器获取了片的所有SSTable文件名，片服务器通过一些索引机制可以知道所需要的数据在哪个SSTable文件，然后从GFS中读取SSTable文件的数据，这个SSTable文件可能分布在好几台chunkserver上。

### 元数据表的结构

元数据表（METADATA table）是一张特殊的表，它被用于数据的定位以及一些元数据服务，不可谓不重要。但是 Bigtable 论文里只给出了少量线索，而对表的具体结构没有说明。这里我试图根据论文的一些线索，猜测一下表的结构。首先列出论文中的线索：

1. The METADATA table stores the location of a tablet under a row key that is an encoding of the tablet's table identifier and its end row.
2. Each METADATA row stores approximately 1KB of data in memory（因为访问量比较大，元数据表是放在内存里的，这个优化在论文的locality groups中提到）.This feature（将locality group放到内存中的特性） is useful for small pieces of data that are accessed frequently: we use it internally for the location column family in the METADATA table.
3. We also store secondary information in the METADATA table, including a log of all events pertaining to each tablet(such as when a server begins serving it).

第一条线索，元数据表的行键是由片所属表名的id和片最后一行编码而成，所以每个片在元数据表中占据一条记录（一行），而且行键既包含了其所属表的信息也包含了其所拥有的行的范围。譬如采取最简单的编码方式，元数据表的行键等于strcat(表名，片最后一行的行键)。

