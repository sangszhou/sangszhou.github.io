---
layout: post
title:  "Java 内存模型"
date:   "2016-09-03 00:00:00"
categories: java
keywords: java, concurrent
---

## 内存模型

顺序一致性内存模型是一个理论参考模型。 JMM 和处理器内存模型在设计时会把顺序一致性模型
作为参照。 但如果完全按照顺序一致性模型来实现处理器和 JMM, 那么很多处理器和编译器的优化
将被禁止, 这对执行性能有很大的响应。


### 线程, 主内存和工作内存

主内存主要对应 JAVA 堆中对象的实例数据部分, 而工作内存则对应于虚拟机栈中的部分区域。

主内存除了保存实例数据外, Java 堆还保存了对象的其他信息, 对于 hotspot 虚拟机来讲, 有 Mark word(对象 hash 码),
GC 标志, GC 年龄, 同步锁等信息), Class point(指向存储类型元信息的指针)以一些用于字节对齐补白的填充数据

### 重排序

在执行程序时为了性能, 编译器和处理器常常会对指令重排序, 重排序分为三种类型:

> 1. 编译器优化重排序。 编译器在不改变单线程程序语义的前提下可以重新安排语句的执行顺序
> 2. 指令级并行的重排序。 现代处理器使用了指令级并行技术来将多条指令重叠执行
> 3. 内存系统重排序。 由于处理器使用缓存和读写缓冲区, 这使得加载和存储操作看上去是乱序的

1 属于编译器重排序, 2,3 属于处理器重排序。 这些重排序都会导致多线程程序内存可见性问题。
对于处理器重排序, JMM 的处理器重排序规则会要求 java 编译器在生成指令序列时,插入特定
类型的内存屏障, 通过内存屏障来禁止特定类型的处理器重排序


### As if serial 语义

不管怎么重排序, 单线程的执行结果不能被改变。当代码存在数据依赖性时, 编译器和处理器不会对存在
数据依赖关系的操作做重排序, 因为这会影响执行结果的正确性。但仅仅限制对数据依赖的重排序
不能保证正确运行的程序。

**重排序对多线程的影响**

```java
class ReorderExample
    int a = 0
    boolean flag = false

public void write
    a = 1    // 1
    flag = true  // 2

pubic void reader
    if(flag)  // 3
        int i = a * a // 4
```

假设有两个线程 A, B. A 首先执行了 write 方法, 随后 B 执行了 reader 方法, 线程 B 在
执行了操作 4 时, 能否看到线程 A 在操作 1 对共享变量 a 的写入?

答案是不一定。
因为 1 和 2 没有数据依赖关系, 编译器和处理器可以对这两个操作重排序, 同样的操作 3, 4 也没有
数据依赖关系, 这两行代码也可以重排序。当 flag = true 现执行时, int i = 0*0。当 int i = a * a 
执行时, 即使 1, 2 没有重排序, 也会使 i = 0 * 0

这里有一点需要注意, 那就是控制依赖能不能防止重排序。当代码存在控制依赖时, 会影响指令执行的并行度。
为此, 编译器和处理器会采用猜测执行来克服控制相关性对并行度的影响。 以处理器的猜测执行为例, 执行
线程 B 的处理器可以提前读取并计算 a * a 并把结果缓存在 temp 变量中, 当 flag 判断通过时, 执行的
是 int i = temp, 所以实际上也是执行了重排序。所以在多线程程序中, 对存在控制依赖的操作重排序可能
改变程序的结果。

编译器, 处理器重排序似乎把一起问题都搞乱了, 那么有什么东西是不变的呢? 程序员怎么保证
自己的多线程程序是线程安全的呢

### happens-before

JSR (java specification request) 133 使用 happens-before 的概念来阐述操作之间的
内存可见性。 在 JMM 中一个操作执行的结果需要对另外一个操作课件, 那么这两个操作之间
必须要有 happens-before ,这两个操作可以是同一个线程内, 也可以是不同线程之间。

**程序顺序规则(Program Order Rule):** 一个线程的每个操作, happens-before 与该线程的任意后续操作

**监视器锁规则(Monitor Lock Rule):** 对一个监视器解锁, Happens-before 与随后对这个监视器加锁

**volatile 规则(Volatile Variable Rule):** 对一个 volatile 域的写, happens-before 对任意后续对这个 volatile 域的读

**传递性(Transitivity):** 如果 A happens-before B, 且 B happens-before C, 那么 A happens before C

线程启动, 线程终止和线程中断规则

两个操作具有 happens-before 并不意味着前一个操作必须在后一个操作之前执行, happens-before 
仅要求前一个操作的执行结果对后一个操作可见, 且前一个操作的顺序排在后一个操作之前

happens-before 是 JMM 提供给程序员写多线程程序时的参考, 按照 happens-before rule 写出的代码
即使发生了重排序, 正确性也是可以保证的。

#### 例子 1

```java
double pi = 3.14 // A
double r = 1.0 // B
double area = pi * r * r // C
```

A happens-before B, B happens-before C, A happens-before C

其中第三个 happens-before 关系是根据传递性推导出来的。

A happens-before B, 但实际执行时, B 可以先于 A, JMM 仅仅要求前一个操作对后一个操作
可见, 这里的操作 A 的执行结果不需要对 B 可见, 重排序的执行结果和正常顺序一致, 所以 JMM 
会认为这种重排序并不非法, JMM 允许这种重排序。

#### 例子 2 

```java
int a = 0
volatile boolean flag = false

public void writer
    a = 1   // 1
    flag = true // 2

public void reader
    if(flag)  // 3
        int i = a  // 4
```

假设线程 A 执行 writer 方法**之后**, 线程 B 执行 reader 方法。

1. 根据次序规则 1 happens 2, 3 happens before 4
2. 根据 volatile 规则, 2 happens before 3
3. 根据传递性, 1 happens-before 4

一个线程时间上的先发生不代表是先行发生的。比如没有通过关键字修饰的 setVal, getVal 分别在两个线程中, 那么时间上的先后不能保证可见

一个线程**先行发生**也不代表它在**线程时间上先发生**, 指令重排序

```java
int i = 1
int j = 1
```

前一条命令先行发生于后一条命令, 但是这两条命令执行的时间是不确定的

###线程状态转换

6 个状态

New: 创建但尚未启动

Runnable: 包括操作系统状态中的 Runnable 和 Ready, 表示他有可能在执行, 也有可能在等待 CPU

Waiting: 等待被显式的唤醒。以下方法会让线程进入无限期等待状态

```java
Object.wait
Thread.join // why?
LockSupport.park
```

我觉得 condition 应该也算的

Timed Waiting: 一段时间后会被线程自动唤醒

```java
Object.wait(time)
Thread.join(time)
LockSupport.parkNano()
LockSupport.parkUntil()
```

Blocked: 线程被阻塞了, 与等待状态的区别是阻塞状态在等待着获取一个排它锁, 这个事件在另一个线程放弃锁的时候发生。
在线程等待进入同步区时, 线程进入这个状态

Terminated: 线程终止

### JMM 的最小安全性

线程执行时读取的值, 要么是之前某个线程写入的值, 要么是默认值 (0, null, false)。JMM 保证不会读到部分值, 为了实现最小
安全性, JVM 在堆上分配对象时, 自己会先清零空间, 然后才会在上面分配对象, JVM 会同步这两个操作。在已清零的内存空间
分配对象时, 初始化已经完成了。(堆上则不然)

JMM 不保证对 64 bit 的 Long 型和 double 型变量读写具有原子性。第三个差异与处理器总线的工作机制相关, 
在计算器中, 数据通过总线在处理器和内存之间传递, 每次处理器和内存之间的数据传递都是通过一系列步骤完成的,
这一系列步骤成为总线事务 (bus transaction)。总线事务包括读写事务两种, 在一个事务发生时, 总线会屏蔽
其他试图并发的总线事务。

### 线程安全的实现方法

**互斥同步**

同步时指多线程并发访问共享数据时, 保证共享数据在同一时刻只被一条(或是一些,使用信号量时)线程使用。

互斥是实现同步的一种手段, 临界区, 互斥量和信号量都是实现互斥的方式。互斥是因, 同步是果, 互斥是方法, 同步时目的。

**非阻塞同步** 

Test and set, Fetch and increment, swap, cas

**无同步方案**

线程本地存储, ThreadLocal, Thread.ThreadLockHashCode 作为主键

### 锁优化

**锁消除**

```java
public String concatString(String s1, String s2, String s3)
    return s1 + s2 + s3
```

由于 String 是不可变的类, 对字符串的连接操作总是通过生成新的 String 对象来进行的, 因此 javac 编译器会对 string 连接自动优化。
JDK5 以后内部实现为 stringBuilder, 其 append 方法有同步块, 如果没有发生逃逸的话, 就会进行锁消除

**轻量级锁**

传统的锁被称为重量级锁, 首先轻量级锁不是用来替换重量级锁的, 它的本意是在多线程竞争的前提下, 减少传统重量锁使用操作系统互斥量产生的性能损耗。
 
Hotspot 虚拟机的对象头有三部分组成, 第一个部分是用于存储对象运行时的数据, 如哈希码, GC 分代年龄, 锁标志位, 官方称为 Mark Word。 
第二部分是指向永久代的指针(类型数据), 第三部分是当对象是数组时, 保存数据的大小

在代码块进入同步时 synchronized(object), 如果同步对象没有被锁定, 虚拟机在当前线程的栈帧创建一个名为锁记录(lock record) 的空间, 用于存储锁对象的 Mark Word.

然后, 虚拟机将对象的 mark work 改成指向 lock record 的指针。如果这个更新动作成功了, 那么这个线程就拥有了该对象的锁, 此时对象头更新锁标志。

释放锁时, 就是把 mark word 再换回来

如果这个时候有线程来竞争锁, 那么轻量级锁就膨胀为重量级锁, 新的线程将被阻塞。可能重量级锁才有阻塞能力吧

**偏向锁**

锁会偏向第一个持有它的线程, 如果期间没有其他对象获取锁, 那么该线程永远不需要再获取锁

第一次获取锁时, 假如锁空闲, 就把 mark word 改成线程 ID, 并置操作位, 以后线程在进入同步区都不需要获取锁。当另外一个线程试图访问锁时,
偏向模式就结束了, 此时根据锁是否处于被锁定状态膨胀到重量级锁或者轻量级锁上

![](/images/posts/javaconcurrent/lightbiaslock.jpg)

### ReentrantLock 与 Synchronized 比较

**等待可中断**

当持有锁的线程长时间不释放时, 等待的线程可以选择放弃等待, 该做其他事情。使用 timed wait 实现? 还是 tryLock?
 
此外, ReentrantLock 提供了可轮询的锁请求，他可以尝试的去取得锁，如果取得成功则继续处理，
取得不成功，可以等下次运行的时候处理，所以不容易产生死锁，而synchronized则一旦进入锁请求要么成功，要么一直阻塞，所以更容易产生死锁

**可实现公平锁**

多个线程在等待锁时, 必须按照申请锁的时间顺序来依次获得锁

**锁绑定多个条件**

ReentrantLock 对象可以绑定多个 Condition 对象, 只要调用 newCondition 就行了。为什么 Condition 要和锁在一起呢?


## volatile

volatile 是 java 虚拟机提供的最轻量级的同步机制, 当一个变量被定义为 volatile 时, 他将具备两种特性,
第一是保证此变量对所有线程的可见性, 当一个线程修改了这个变量的值时, 新值对其他线程来说是可以理解得知的。而
普通变量无法做到这一点,变量值在线程之间传递需要通过主内存来完成, 而线程和主内存之间还有工作内存来缓冲数据。
但需要注意的是, volatile 的符合操作不是原子的。

volatile 的第二个语义是禁止指令重排序。编译器为了性能会进行指令重排序。

volatile 的同步机制性能要优于锁 (synchronized 和 concurrent 包下的锁), 但虚拟机对锁实行了许多消除和优化,
很难讲到底快了多少。volatile 的读操作性能消耗和普通变量几乎没什么差别, 写操作因为要在本地代码插入内存屏障来保证处理器不发生乱序执行
, 还是要慢一些的

### 内存模型中的原子性可见性和有序性

**原子性** JMM 直接保证原子性变量操作包括  read, load, assign, use, store 和 write 这六个, 还有 lock 之间操作, 
它的实现是高层字节码 monitorenter 和 monitorexit 

**可见性** 除了 volatile 外, 还有两个关键字能够实现可见性, 他们分别是 synchronized 和 final. final 的可见性是指被
final 修饰的字段,在构造器中一旦被初始化完成, 并且构造器没有把 this 的引用传递出去, 那么其他线程就能看到 final 字段的值。

```java
public static final int i
public final int j

static { i = 0 }
static { j = 0 }
```

this 指针逃逸是一个很危险的事情, 因为其他线程可以通过 this 指针访问到初始化一半的对象

**有序性** 线程内表现为串行有序的, 指令重排序和工作内存同步延迟导致线程之间无序。



### Volatile 的内存语义

* 当写一个 volatile 变量时, JMM 会把该线程对应的本地内存中的共享变量值刷新到主内存
* 当读一个 volatile 变量时, JMM 会把该线程对应的本地内存中的共享变量置为无效

### Volatile 内存语义的实现



## 参考
[并发编程网](http://ifeve.com/java-memory-model-0/)