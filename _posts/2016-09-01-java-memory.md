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

### **CMS 收集器**

并发低停顿收集器


