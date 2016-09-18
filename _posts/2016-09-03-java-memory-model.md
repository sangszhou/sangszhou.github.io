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

**程序顺序规则: ** 一个线程的每个操作, happens-before 与该线程的任意后续操作
**监视器锁规则: ** 对一个监视器解锁, Happens-before 与随后对这个监视器加锁
**volatile 规则: ** 对一个 volatile 域的写, happens-before 对任意后续对这个 volatile 
域的读
**传递性: ** 如果 A happens-before B, 且 B happens-before C, 那么 A happens before C

两个操作具有 happens-before 并不意味着前一个操作必须在后一个操作之前执行, happens-before 
仅要求前一个操作的执行结果对后一个操作可见, 且前一个操作的顺序排在后一个操作之前。

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

### JMM 的最小安全性

线程执行时读取的值, 要么是之前某个线程写入的值, 要么是默认值 (0, null, false)。JMM 保证不会读到部分值, 为了实现最小
安全性, JVM 在堆上分配对象时, 自己会先清零空间, 然后才会在上面分配对象, JVM 会同步这两个操作。在已清零的内存空间
分配对象时, 初始化已经完成了。(堆上则不然)

JMM 不保证对 64 bit 的 Long 型和 double 型变量读写具有原子性。第三个差异与处理器总线的工作机制相关, 
在计算器中, 数据通过总线在处理器和内存之间传递, 每次处理器和内存之间的数据传递都是通过一系列步骤完成的,
这一系列步骤成为总线事务 (bus transaction)。总线事务包括读写事务两种, 在一个事务发生时, 总线会屏蔽
其他试图并发的总线事务。

## Volatile

### Volatile 的内存语义

* 当写一个 volatile 变量时, JMM 会把该线程对应的本地内存中的共享变量值刷新到祝内存
* 当读一个 volatile 变量时, JMM 会把该线程对应的本地内存中的共享变量置为无效

### Volatile 内存语义的实现



## 参考
[并发编程网](http://ifeve.com/java-memory-model-0/)