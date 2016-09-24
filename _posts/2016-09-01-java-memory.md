---
layout: post
title:  "memory model and gc"
date:   "2016-09-02 00:00:00"
categories: java
keywords: java, io
---

@todo 内存泄漏应该放在这里

## Java 内存划分

![](/images/posts/javamem/memdis.png)

**线程私有的部分:**

> 1. 程序计数器
   当前线程所执行的字节码行号指示器
> 2. java 虚拟机栈
   java 方法执行的的内存模型, 每个方法被执行时都会创建一个栈帧, 存储局部变量表, 操作栈, 动态链接
   方法出口等信息
   每个线程都有自己独立的栈空间, 线程栈只存基本类型和对象地址, 方法中局部变量在线程空间
> 3. 本地方法栈
   Native 方法服务。在 Hotspot 虚拟机中和 java 虚拟机栈合二为一

**线程共享的部分:**

> 1. java 堆
   存放对象的实例, 几乎所有对象的实例和属性都在这
> 2. 方法区
   存储已经被虚拟机加载的类信息, 常量, 静态变量, JIT 编译后的代码
> 3. 运行时常量池
   方法区的一部分, 用于存放编译期产生的各种字节面量和符号引用 (what's 符号引用)
> 4. 直接内存
   NIO, Native 函数直接分配堆外内存。DirectBuffer 也会引用此部分内存

**对象访问:**

对象访问分为通过句柄访问和直接指针两种形式。 通过句柄直接访问是指 Java 栈中的 reference
指向的是 java 堆中的句柄池, 句柄池中包含两种指针类型, 分别是到对象实例数据的指针和到
对象类型数据的指针, 其中到对象类型数据在方法区, 对象实例数据也在 java 堆

通过直接指针访问就简单些, reference 指向的位置就是对象实例数据的开始位置, 对象实例数据
中包含指向方法区的指针

## 内存溢出

内存溢出分为虚拟机栈和本地方法栈溢出, java 堆溢出, 运行时常量区溢出和方法区溢出四种

### 虚拟机栈和本地方法溢出

`StackOverflowError` 线程请求栈深度大于虚拟机所允许的最大深度 (递归函数)
`OutOfMemoryError` 虚拟机在扩展栈时无法申请到更多的内存, 一般可以通过不停的创建线程
一起这种问题

### java 堆溢出
创建大量的对象并且生命周期都很长的情况下会 OOM

### 运行时常量区溢出
OOM PermGen space. 典型的例子就是 String.intern 方法

### 方法区溢出
方法区存放 class 等元信息, 如果产生大量的类(使用 cglib) 就会引起溢出

## 垃圾收集

### GC 算法

**标记清除**:
首先标记出所有需要回收的对象, 在标记完成后统一回收。效率不高, 且会出现内存碎片

**复制算法**:
将内存分为 from, to 两个区域, 垃圾回收时把 from 区存活的对象移动到 to 区
新生代中的对象大部分存活期很短, 可以把内存分为较大的 eden 空间和两块较小的 survivor
空间, 每次使用 eden 和其中的一块 survivor。回收时, 把 eden 和一块 survivor 存活
的对象拷贝到另一块 survivor 区。默认的 eden 和 survivor 是 8:1。回收时,当 survivor
的空间不够用时, 需要依赖其他内存(老年代)进行分配担保。因为没有内存区为老年代分配担保,
所以老年去不能用复制算法
 
**标记整理算法**:
让所有存活的对象向一端移动,然后直接清理掉边界以外的内存

**分代收集算法**:
把 java 堆分成新生代和老年代, 根据各代的特点选取最适当的收集算法。在新生代, 每次垃圾收集
仅有少量对象存活, 那就算复制算法。老年代存活率高, 没有额外空间做担保, 就必须用标记清理和
标记整理算法

![](/images/posts/javamem/visualvm.png)

这是 java8 程序的 visual vm 的可视化界面, 从图中可以看出 Eden 和 survior 的比例恰好是 8:2, 
Eden 和 Old Gen 每次 GC 使用的时间却差不多, 都是 55ms 左右, 所以 full gc 也没多用
多少时间。Metaspace 是 Java8 的方法区, 竟然有 1G 大小。

### 垃圾收集器

**何时进行 minor GC, major GC?**

> 对象在 Eden Space 完成内存内存分配, 当 Eden space 满了,再创建对象会因为申请不到空间触发
> minor GC, 进行新生代的垃圾回收
> Minor GC 时, Eden Space 和 Survior 不能回收的对象放到另一个 Survivor 区
> 如果发现另一个 survivor 也满了, 这些对象被 copy 到 old 区, 或者 survivor 并没有满,
> 但是对象已经够 old 了, 也被放到 old Gen
> 当 old space 满了时, 进行 full GC

## **Hotspot 的垃圾收集器**

![](/images/posts/javamem/gc_collector_hotspot.jpg)

### **Serial 收集器**

1. --XX:+UseSerialGC 参数打开此收集器
2. Client 模式下新生代的默认收集器 (?)
3. 较长的 Stop the world 时间
4. 使用标记复制算法, 简单高效

### **ParNew 收集器**

1. -XX:+UseParNewGC 启动, --XX:ParallelGCThreads 指定线程数目
2. Serial 收集器的多线程版本
3. 默认线程数和 CPU 数目相同

对比 Serial 收集器

![](/images/posts/javamem/serial.png)

### **Parallel Scavenge**

1. 关注的是可控制的吞吐量
2. 新生代并行收集器, 采用 COPY 算法
3. Server 模式默认新生代收集器

### **Serial Old 收集器**

1. Serial 的老年代版本
2. Client 模式的默认老年代收集器
3. CMS 收集器的后备预案 
4. --XX:+UseSerialGC 开启此收集器

### **Parallel Old 收集器**

1. --XX:+UseParallelGC -XX:+UseParallelOldGC 启动此收集器
2. Server 模式的默认老年代收集器
3. Parallel Scavenge 的老年代版本, 使用多线程和 mark-sweep 算法
4. 一般使用 Parallel Scavenge + Parallel old 达到大吞吐量

### **CMS 收集器. 并发低停顿收集器**

**并行:** 指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态；

**并发:** 指用户线程与垃圾收集线程同时执行

CMS收集器是一款并发收集器，是一种以获取最短回收停顿时间为目标的收集器，它是基于标记–清除算法实现的，它整个过程包含四个有效的步骤：

1) 初始标记（CMS initial mark）

2) 并发标记（CMS concurrent mark）

3) 重新标记（CMS remark）

4) 并发清除（CMS concurrent sweep）

其中，初始标记、重新标记仍然需要“Stop the World”，但是它们的速度都很快。初始标记只是标记一下GC Roots能直接关联到的对象，
速度很快，并发标记阶段就是进行GC Roots

Tracing的过程，重新标记是为了修正并发标记期间，因用户线程继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始化标记阶段稍长一些，但远比并发标记的时间短。

由于整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，所以总体上来说，
CMS收集器的内存回收过程是与用户线程一起并发的执行的

![](/images/posts/javamem/cms.png)

CMS的主要优点是并发收集、低停顿，也称之为并发收集低停顿收集器（Concurrent Low Pause Collector），其主要缺点如下：

> CMS回收器采用的基础算法是 Mark-Sweep。所有CMS不会整理、压缩堆空间。这样就会有一个问题：
> 经过CMS收集的堆会产生空间碎片。 CMS不对堆空间整理压缩节约了垃圾回收的停顿时间，但也带来的堆空间的浪费。
> 为了解决堆空间浪费问题，CMS回收器不再采用简单的指针指向一块可用堆空 间来为下次对象分配使用。而是把一些未分配的空间汇总成一个列表，
> 当JVM分配对象空间的时候，会搜索这个列表找到足够大的空间来hold住这个对象

> 需要更多的CPU资源。从上面的图可以看到，为了让应用程序不停顿，CMS线程和应用程序线程并发执行，这样就需要有更多的CPU，
> 单纯靠线程切 换是不靠谱的。并且，重新标记阶段，为空保证STW快速完成，也要用到更多的甚至所有的CPU资源。当然，多核多CPU也是未来的趋势

> CMS的另一个缺点是它需要更大的堆空间。因为CMS标记阶段应用程序的线程还是在执行的，那么就会有堆空间继续分配的情况，
> 为了保证在CMS回 收完堆之前还有空间分配给正在运行的应用程序，必须预留一部分空间。也就是说，CMS不会在老年代满的时候才开始收集。
> 相反，它会尝试更早的开始收集，已避免上面提到的情况: 在回收完成之前，堆没有足够空间分配！默认当老年代使用68%的时候，
> CMS就开始行动了。 – XX:CMSInitiatingOccupancyFraction =n 来设置这个阀值

### Java itself

**JRE: java runtime environment**  java 运行环境

JRE是运行java所需要的环境。包含JVM标准实现和JAVA核心类库，以及 javaplug-in。
可以在JRE上进行运行、测试和传输应用程序。JRE不包括编译器，调试器和其他工具。
也就是说，如果直接运行一个java编译好了的 class 文件，使用JRE就OK 了。

但是如果你要开发一个java文件，然后对它进行编译，调试等工作，这个时候就要用到JDK 了。

**JDK: java development kit**

> javac：java的编译器，可以将java文件编译成字节码

> java： java的解释器，可以将字节码进行解释运行

> jdb:   java 调试器，可以设置断点和检查变量，逐行运行程序

> javap：java反汇编器，显示编译类文件中的可访问功能和数据，同时显示字节码的含义

> jstat: 对jvm的内存使用量进行监控

> jmap - Memory Map: Prints shared object memory maps or heap memory details of a given JVM process or a Java core file on the local machine or on a remote machine through a debug server。
  jmap 能够打印出jvm的内存使用详情

> jconsole: Java进行系统调试和监控的工具

> jstack 

**Java client 和 server 模式**

```
java -version //查看JVM默认的环境   
java -client -version //查看JVM的客户端环境,针对GUI优化,启动速度快,运行速度不如server   
java -server -version //查看JVM的服务器端环境，针对生产环境优化,运行速度快，启动速度慢   
```

**常用配置**

```
-server
-Xms2048m
-Xmx1224m
-XX:PermSize=64M
-Xss512k //每个线程的Stack大小
-verbose:gc 
-XX:+UseConcMarkSweepGC ：缩短major收集的时间
-XX:+UseParNewGC ：缩短 minor 收集的时间
```