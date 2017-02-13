---
layout: post
title: Java thread dump analysis
categories: [java]
description: java
keywords: java, thread
---

## 线程状态分析

### stacktrace 解读

```
"Worker-15" prio=10 tid=0x09b7b400 nid=0x12f2 in Object.wait() [0xb30f9000]    
   java.lang.Thread.State: TIMED_WAITING (on object monitor)    
    at java.lang.Object.wait(Native Method)    
    at org.eclipse.core.internal.jobs.WorkerPool.sleep(WorkerPool.java:188)    
    - locked <0x6e5f87c0> (a org.eclipse.core.internal.jobs.WorkerPool)    
    at org.eclipse.core.internal.jobs.WorkerPool.startJob(WorkerPool.java:220)    
    at org.eclipse.core.internal.jobs.Worker.run(Worker.java:50)  
```

Thread name: worker-15

线程优先级 prio = 10

java 线程的 identifier -x09

native 线程 identifier: nid = 0x12f2

线程的状态 Object.wait() [0xb30f9000]   java.lang.Thread.State: TIMED_WAITING (on object monitor)  

线程栈起始地址 0xb30f9000

JRE 包括 Java API classes 和 JVM

### **JVM 线程状态**

状态有 `Runnable`, `Wait on condition`, `Waiting for monitor entry`, `in Object.wait`

Wait on condition: 该状态出现在线程等待某个条件的发生，具体什么原因要结合 stacktrace 分析，最常见的是等待忘了读写，如果网络数据没有准备好，线程就会等待在那里，另一种出现 Wait on condition 的常见情况是线程在 sleep, 等待 sleep 的时间到时，将被唤醒。

此时线程的状态大概分为几种：1 waiting(parking) 一直等待那个条件的发生 2. Timed_waiting(parking 或者 sleeping) 定时的，那个条件超时时也会唤醒自己


Wait for monitor entry 和 Object.wait(), 这两种情况不太出现，Java 会通过 Monitor 来实现线程互斥和写作，可以把它理解成一个对象或者 class 锁拥有的锁，每个对象和 class 有且仅有一个

![java_monitor](/images/posts/javavm/java_monitor.jpg)

每个 Monitor 在每个时刻只能被一个线程拥有，拥有 monitor 的线程叫做 active thread, 而其他线程都叫做 waiting thread, 分别在两个队列 Entry set 和 Wait set 里面等待。在 Entry set 中等待的线程状态是 waiting for monitor entry, 而 wait set 中等待的状态是 in Object.wait() 当线程获得 montior 但发现继续运行的条件无法满足时，它调用对象的 wait 方法，放弃 monitor 进入 wait set 队列

一般都是 RMI 相关线程（RMI renewClean, GC daemon, RMI reaper) GC 线程 (finalizer), 引用对象垃圾回收线程 (reference handler)

在 Entry set 中的线程都等待都拿到 monitor, 拿到线程就成了 runnable 线程，否则就会一直处在 wait for monitor entry 

**Demo1**

```java
public class MyThread implements Runnable {
    @Override
    public void run() {
        synchronized (this) {
            for(int i = 0; i < 1; i --) {
                System.out.println(Thread.currentThread().getName() + " synchronized loop " + i);
            }
        }
    }

    public static void main(String args[]) {
        MyThread t1 = new MyThread();
        Thread ta = new Thread(t1, "A");
        Thread tb = new Thread(t1, "B");

        ta.start();
        tb.start();
    }
}

// stack trace

"B" prio=10 tid=0x0969a000 nid=0x11d6 waiting for monitor entry [0x8bb22000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at org.marshal.MyThread.run(MyThread.java:7)
	- waiting to lock <0x94757078> (a org.marshal.MyThread)
	at java.lang.Thread.run(Thread.java:636)

"A" prio=10 tid=0x09698800 nid=0x11d5 runnable [0x8bb73000]
   java.lang.Thread.State: RUNNABLE
	at java.io.FileOutputStream.writeBytes(Native Method)
	at java.io.FileOutputStream.write(FileOutputStream.java:297)
	at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
	at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
	- locked <0x947571b0> (a java.io.BufferedOutputStream)
	at java.io.PrintStream.write(PrintStream.java:449)
	- locked <0x94757190> (a java.io.PrintStream)
	at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:220)
	at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:290)
	at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:103)
	- locked <0x947572a0> (a java.io.OutputStreamWriter)
	at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:185)
	at java.io.PrintStream.write(PrintStream.java:494)
	- locked <0x94757190> (a java.io.PrintStream)
	at java.io.PrintStream.print(PrintStream.java:636)
	at java.io.PrintStream.println(PrintStream.java:773)
	- locked <0x94757190> (a java.io.PrintStream)
	at org.marshal.MyThread.run(MyThread.java:8)
	- locked <0x94757078> (a org.marshal.MyThread)
	at java.lang.Thread.run(Thread.java:636)
```

在 stacktrace 中， 0x94757078 就是两个线程争夺的监视器

在 stacktrace 中，会自动分析死锁问题，所以死锁反而不是个问题

### **Java 中的其他线程**

**Hotspot GC Thread**

```
"GC task thread#0 (ParallelGC)" prio=10 tid=0x08726800 nid=0x1e30 runnable  
  
"GC task thread#1 (ParallelGC)" prio=10 tid=0x08727c00 nid=0x1e31 runnable  
```

当虚拟机打开了 parallel GC, 这些线程就会定期清理内存。

**HotSpot VM Thread**

```
"Low Memory Detector" daemon prio=10 tid=0x097eb000 nid=0xa9f runnable [0x00000000]    
   java.lang.Thread.State: RUNNABLE    
    
"CompilerThread0" daemon prio=10 tid=0x097e9000 nid=0xa9e waiting on condition [0x00000000]    
   java.lang.Thread.State: RUNNABLE   

```
JVM 管理的内部线程，这些内部线程都是来执行 native 操作的

**另外一个例子**


```
"RMI TCP Connection(idle)" daemon prio=10 tid=0x00007fd50834e800 nid=0x56b2 waiting on condition [0x00007fd4f1a59000]
   java.lang.Thread.State: TIMED_WAITING (parking)
at sun.misc.Unsafe.park(Native Method)
- parking to wait for  <0x00000000acd84de8> (a java.util.concurrent.SynchronousQueue$TransferStack)
at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:198)
at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:424)
at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:323)
at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:874)
at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:945)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:907)
at java.lang.Thread.run(Thread.java:662)
```

1) “TIMED_WAITING (parking)”中的 timed_waiting 指等待状态，但这里指定了时间，到达指定的时间后自动退出等待状态；parking指线程处于挂起中。

2) “waiting on condition” 需要与堆栈中的 “parking to wait for  <0x00000000acd84de8> (a java.util.concurrent.SynchronousQueue$TransferStack)”结合来看。首先，本线程肯定是在等待某个条件的发生，来把自己唤醒。其次，SynchronousQueue 并不是一个队列，只是线程之间移交信息的机制，当我们把一个元素放入到 SynchronousQueue 中时必须有另一个线程正在等待接受移交的任务，因此这就是本线程在等待的条件。



### Wait on condition

等待另一个条件的发生，来把自己唤醒

### 自己遇到过的问题

**CPU 使用率高的问题**

前两天有个同事跑来找我，说 SOA 的 CPU 利用率接近 100%， 要我看一看，那天正好是周一，周一上午的 weekly deployment 有可能引入 bug 导致 CPU 利用率飙升，所以要确认是因为要处理的任务太多还是因为代码里有 bug.

以前，debug 都是靠 Visual VM, 但是前段时间，老大把 production 的访问权限全部 revoke 了，所以只能用 Jstack 这种比较原始的工具来查看 trace, 找到进程的 PID 然后 jstack $PID，在 stacktrace 中，主要关注的点并不是 Blocked 或者 Waiting, 而是那些处于 Runnable 的线程，从中找到两大类线程，一类是 Netty 一类是 Tika, Netty 目前好像只有 ES client 在用，大量处于 Runnable 的 Netty 线程似乎是有些不正常，我觉得既然是 Networking 相关的进程，状态应该是 Timed_waiting 才对。更令我担忧的是 Tika 线程，前段时间有个实习生提交了对 tika 处理逻辑的优化，按说应该减少 CPU 的使用率，但是结果却恰恰相反，我认为 Tika 出问题的可能性更大些，所以进一步研究了 tika 的问题

首先，我想知道 tika 到底占用了多少的 CPU 资源。先用 TOP -H -p $PID 找到 Java 进程对应的所有线程，下面是以前总结的查看 CPU 利用率最高线程的脚本

```shell
#!/bin/bash

if [ $# -eq 0 ];then
    echo "please enter java pid"
    exit -1
fi

pid=$1
jstack_cmd=""

if [[ $JAVA_HOME != "" ]]; then
    jstack_cmd="$JAVA_HOME/bin/jstack"
else
    r=`which jstack 2>/dev/null`
    if [[ $r != "" ]]; then
        jstack_cmd=$r
    else
        echo "can not find jstack"
        exit -2
    fi
fi

#line=`top -H  -o %CPU -b -n 1  -p $pid | sed '1,/^$/d' | grep -v $pid | awk 'NR==2'`

line=`top -H -b -n 1 -p $pid | sed '1,/^$/d' | sed '1d;/^$/d' | grep -v $pid | sort -nrk9 | head -1`

echo "$line" | awk '{print "tid: "$1," cpu: %"$9}'

tid_0x=`printf "%0x" $(echo "$line" | awk '{print $1}')`

$jstack_cmd $pid | grep $tid_0x -A20 | sed -n '1,/^$/p'
```

其中 sort -nrk9 表示按照第9列的大小从大到小排序，第九列的元素都是 int 类型

grep xxx -A20 表示打印出 xxx 和紧跟着 xxx 的 20 行 log

当时我并没有完全按照这个脚本来做，而是大致看了下 SOA 里占用 CPU 资源比较多的线程到底跑了多久，我看到最长的一个已经超过 115 min, 也就是说问题的原因很有可能就是出现在 tika 身上，随便找了一个 CPU 占用率比较高的线程，把线程 ID 从 十进制转到 16 进制到 stacktrace 中查看他的调用链，结果发现 tika 相关的调用链都卡在一个递归调用函数上

为了从根本上确认问题，我们关了上游的 storm，停止处理新的文件，过了一段时间，CPU 利用率全部降下来了，这说明两个问题第一个是代码只是处理的蛮，没有死循环或者 deadlock, 第二个是问题的根源就是新的代码有问题。

从脚本的角度，我学到了一点知识，就是如何仅从 stacktrace 的角度查看线程的状态。jstack 只会进程的一个 snatshot, 它并不是一段时间的信息，为了看到一段时间的信息，我们可以用 watch 命令，结合 grep $tid -A20， 

**消息处理失败的问题**

在处理完上一个问题的第二天，又发生了某些消息在后台逻辑处理失败的问题。当 OPS 找到我时，我的第一反应是难道昨天那个问题又出现了？因为 CPU 使用率太高导致自己的另外一些 Service 得不到处理？从问题的状况来看，并不是这样，因为前端在操作完毕后很快就收到报错信息。从 Log 中查看原因，发现是 Mysql 数据库连接不上。数据库连接不上无外乎数据库宕机或者数据库的用户名密码不对，但这两个猜测都很快被否定了，因为从监控上看，数据库运行正常，配置文件都是从配置中心获取的，也不应该有问题。这时候 OPS team 告诉我，**有可能是数据库的连接数过多导致的**，然后他跑到数据库的机器上，运行了 netstat 命令，查看连接数，大约 500 左右，又分析了连接的来源, 把 IP 相同的连接聚集到一起，发现有一个机器的连接数是 150，大大超出了正常的范围，因为我在代码中配置的连接数是 10，重启了那台机器，就一起正常了，这可能是第三方库不够稳定的原因，至于到底出现了什么问题也无从查起。


**很多 service 响应慢问题**

这是今年上半年的一个问题，从这个问题中，team 中的所有成员都会 Akka 的立即提升了一个档次。问题是这样的，OPS team 告诉我某些 API 响应时间很慢，有些甚至超过了一分钟。我觉得很奇怪，那天不是周一，所以不会有新的代码上线，而在没有代码变化的情况下出现问题肯定就是环境导致的。这个环境问题指的是两点，第一点是定时任务，第二点是业务逻辑的请求过多，系统处理不过来了。询问了 team 的所有 dev 后，这两个问题都被排除了。只有看 stacktrace 了，那时候，SOA 还连着 visual vm, 看 stacktrace 是相当方便的，我看了一会，没有发现任何问题，stacktrace 的线程数是 500 左右，正常，大部分状态时 waiting, runnable, 这也正常，我其实挺希望看到处于 block 状态的线程的，因为找到它就能确认系统变慢是死锁问题。

重启，发现重启以后的前十分钟整个系统工作正常，十分钟后 service 又开始变卡，直至响应超时。

继续看 visual vm, 这次是长时间的看，我发现，每隔一段时间，会有一段红色的线程闪过，红色就是 blocked, 因为闪的速度太快，我往往来不及看到这个线程的调用链就消息了，不过好消息是，我知道系统内有锁，这个锁会让某些线程 block 一段时间。这次 stacktrace 的查看让我意识到一个很重要的问题，那就是我们的线程都是没有名字的，即便是看到了某个线程处于比较危险的状态，也不知道这个线程到底是谁创建的。在后面的编码中，**一定要给线程起名字**。

持续观察，直到看到红色线程的 Stacktrace 是某个 app 的处理逻辑，我找到这个 app 的负责人，问他是不是哪里有问题，或者是否用了很多的锁。1小时候，他告诉我，处理文件时，的确用到了锁。出现问题的原因不能完全确认，但是至少这个点我们可以尝试 fix 再查看其效果。以前，我们整个 component 都共用一个线程池，我把 app 放到一个单独的线程池中，这个线程池的线程数目少于 CPU 的个数，上了 hotfix，观察线程的使用状态，发现 app thread pool 已经在 work 了，而其他的 service 都处于正常的状态。至此，问题解决。后续，我们对每个 service 都起一个线程池，每个持久层也有一个线程池，如果一个 service 用到了持久层，那么用哪个线程池是随意的，此外给线程池起名字，这样一旦出现的线程问题，仅从线程的名字上就能得到很多信息。