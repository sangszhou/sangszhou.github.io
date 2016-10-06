---
layout: post
title: java concurrent 3
categories: [java]
keywords: java, concurrent
---

### 1)现在有 T1、T2、T3 三个线程，你怎样保证T2在T1执行完后执行，T3在T2执行完后执行

这个线程问题通常会在第一轮或电话面试阶段被问到，目的是检测你对”join”方法是否熟悉。这个多线程问题比较简单，可以用join方法实现

### 2)在Java中Lock接口比synchronized块的优势是什么？

你需要实现一个高效的缓存，它允许多个用户读，但只允许一个用户写，以此来保持它的完整性，你会怎样去实现它？

**Lock 比 synchronized 的优势**

1. 它允许以一种更灵活的方式来构建synchronized块。使用synchronized关键字，你必须以结构化方式得到释放synchronized代码块的控制权。
   Lock接口允许你获得更复杂的结构来实现你的临界区。
2. Lock 接口比synchronized关键字提供更多额外的功能。新功能之一是实现的tryLock()方法。这种方法试图获取锁的控制权并且如果它不能获取该锁，
   是因为其他线程在使用这个锁，它将返回这个锁。使用synchronized关键字，当线程A试图执行synchronized代码块，如果线程B正在执行它，
   那么线程A将阻塞直到线程B执行完synchronized代码块。使用锁，你可以执行tryLock()方法，这个方法返回一个 Boolean值表示，是否有其他线
   程正在运行这个锁所保护的代码。
3. 当有多个读者和一个写者时，Lock接口允许读写操作分离 (ReadWriteLock)
4. Lock接口比synchronized关键字提供更好的性能
5. 一个 Lock 可以 attach 上多个 condition
6. synchronized在获锁的过程中是不能被中断的
7. 实现公平锁, 所谓的公平，指的是在申请获取锁的队列中，排在前面的线程总是优先获得需要的锁
   举个例子，线程A和B都尝试获得C持有的锁，当C释放该锁时，A和B谁能获得该锁是不确定的，也就是非公平的，而ReentrantLock提供公平地，即先来后到地获取锁的方式。
    


Lock 的实现

ReadLock, WriteLock, ReentrantLock, ReadWriteLock, StampedLock(Jdk1.8)

### ReadWriteLock

**Read Lock:** If no threads have locked the ReadWriteLock for writing, and no thread have requested a write lock 
(but not yet obtained it). Thus, multiple threads can lock the lock for reading.

**Write Lock:** If no threads are reading or writing. Thus, only one thread at a time can lock the lock for writing.

Read Write 的定义就和课本上读写者的问题相同

举个例子, 
Here is a code sketch showing how to perform lock downgrading after updating a 
cache (exception handling is particularly tricky when handling multiple locks in a non-nested fashion):

```java
class CachedData {
  Object data;
  volatile boolean cacheValid;
  final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

  void processCachedData () {
    rwl.readLock().lock();
    if (!cacheValid) {
      // Must release read lock before acquiring write lock
      rwl.readLock().unlock();
      rwl.writeLock().lock();
      try {
        // Recheck state because another thread might have
        // acquired write lock and changed state before we did.
        if (!cacheValid) {
          data =...
          cacheValid = true;
        }
        // Downgrade by acquiring read lock before releasing write lock
        rwl.readLock().lock();
      } finally {
        rwl.writeLock().unlock(); // Unlock write, still hold read
      }
    }

    try {
      use(data);
    } finally {
      rwl.readLock().unlock();
    }
  }
}
```

ReentrantReadWriteLocks can be used to improve concurrency in some uses of some kinds of Collections. 
This is typically worthwhile only when the collections are expected to be large, accessed by 
more reader threads than writer threads, and entail operations with overhead that outweighs synchronization 
overhead. For example, here is a class using a TreeMap that is expected to be large and concurrently accessed.

```java
class RWDictionary {
  private final Map < String, Data > m = new TreeMap < String, Data > ();
  private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
  private final Lock r = rwl.readLock();
  private final Lock w = rwl.writeLock();

  public Data get(String key) {
    r.lock();
    try {
      return m.get(key);
    }
    finally {
      r.unlock();
    }
  }
  public String[] allKeys() {
    r.lock();
    try {
      return m.keySet().toArray();
    }
    finally {
      r.unlock();
    }
  }
  public Data put(String key, Data value) {
    w.lock();
    try {
      return m.put(key, value);
    }
    finally {
      w.unlock();
    }
  }
  public void clear() {
    w.lock();
    try {
      m.clear();
    }
    finally {
      w.unlock();
    }
  }
}
```

**实现**

ReentrantLock 的实现比较简单, 它重写了 AQS 的 tryAcquire 和 tryRelease 方法, 初始状态 (state) 为 0, 一旦被 acquire, state 的
状态就 +1, 被 release state -1. 在 acquire 时还要判断当前线程是否已经获得锁了, 如果已经获得了, 直接访问同步区代码, 但是 state 还是
要 +1. release 时判断自己是否是锁的持有者, 不是的话报错, 是的话返回 true。因为 ReentrantLock 是 exclusive 获取锁的, 所以每次 tryAcquire
和 tryRelease 都是仅获取一次锁。FairSync 和 NonFairSync 的区别就是当 state == 0 时, 是直接获取锁, 还是让队列中的线程拿锁

Read, Write 公用一个 Sync(AQS) 锁, 所以状态(state) 是共享的

```java
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```

**WriteLock**

WriteLock 的实现简单些, 它的 release 和 acquire 分别对应 tryRelease 和 tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState(); // read write 公用的 state
    int w = exclusiveCount(c); // write number
    
    if (c != 0) { // 锁已经被持有
        // 从下面的判断可以看出, readLock 不能提升为 write lock, 因为 w == 0 时(是读锁), 直接返回 false
        if (w == 0 || current != getExclusiveOwnerThread()) // 锁被 ReadThread 持有 或者自己不是写锁的持有者
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires); // 当前状态加上 acquire
        return true;
    }
    
    // 锁没有被任何线程锁持有
    if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
        return false;
    
    setExclusiveOwnerThread(current);
    return true;
```

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively()) throw new IllegalMonitorStateException(); // 不是被自己持有
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;        
```

release 操作就简单的多了, 就和 Reentrant lock 一样了

The API for ReentrantReadWriteLock states that downgrading from a write lock to a read lock is possible, 
but upgrading is not. Which means it's not possible to get a write lock while you have a read lock; 
you need to release the read first. This makes sense - what if you had two threads with read locks, 
and both tried to upgrade? Neither could acquire the write lock until the other gave up the read lock, and that 
wouldn't happen unless the coder explicitly unlocked the read lock before acquiring the write lock. In which 
case it's not really an upgrade, it's a read lock followed by no lock followed by a write lock. That's just what you have to do.

**ReadLock**

一个线程可以同时读写锁, 所以要设置 firstReader ?

HoldCount 的用处是什么? exclusive 和 shared state 已经可以描述当前的锁所处的状态了, 要 HoldCount 何用?

```java

* The number of reentrant read locks held by current thread.
* Initialized only in constructor and readObject.
* Removed whenever a thread's read hold count drops to 0.        
private transient ThreadLocalHoldCounter readHolds;

protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // when exclusiveCount(c) == 0 and 
    if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current) // writer exist
        return -1;
        
    int r = sharedCount(c);
    if (!readerShouldBlock() && r < MAX_COUNT && compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current; // 0->1 的线程称为 firstReader
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {            
            firstReaderHoldCount++; // 为什么要保存这个变量? 因为第一个线程可读写吗?
        } else {
            HoldCounter rh = cachedHoldCounter; // cachedHoldCounter 用处?
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get(); // 设置 cached
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++; }
        return 1;}
    
    return fullTryAcquireShared(current);
``` 

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
        } else if (readerShouldBlock()) {
            if (firstReader == current) {
        } else {
            if (rh == null) {
                rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current)) {
                    rh = readHolds.get();
                    if (rh.count == 0) readHolds.remove();
            if (rh.count == 0) return -1;
        if (sharedCount(c) == MAX_COUNT) throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) { // 成功获取锁
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            if (rh == null)
                rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
            cachedHoldCounter = rh; // cache for release
        return 1;
```


firstReader is the first thread to have acquired the read lock. firstReaderHoldCount is firstReader's hold count.



```java
Thread firstReader = null;
int firstReaderHoldCount;

protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        if (firstReaderHoldCount == 1) // 自己是 first 且自己要退出了
            firstReader = null;
        else firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter; // 这两行代码是为了获取 Hold, cached 是为了不用去查看 ThreadLocal
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
            }
        --rh.count;
        
        for (;;) {
            int c = getState();
            int nextc = c - SHARED_UNIT; // 为什么要减去 SHARED_UNIT, 因为最低位是 exlusive 状态位
            if (compareAndSetState(c, nextc))
                // Releasing the read lock has no effect on readers,
                // but it may allow waiting writers to proceed if
                // both read and write locks are now free.
                return nextc == 0;
        }
```

lock接口在多线程和并发编程中最大的优势是它们为读和写分别提供了锁，它能满足你写像ConcurrentHashMap这样的高性能数据结构和有条件的阻塞。
Java线程面试的问题越来越会根据面试者的回答来提问。我强烈建议在你去参加多线程的面试之前认真读一下Locks
因为当前其大量用于构建电子交易终统的客户端缓存和交易连接空间

### 3)在java中 wait 和 sleep 方法的不同？

通常会在电话面试中经常被问到的Java线程面试问题。

1. 最大的不同是在等待时wait会释放锁，而sleep一直持有锁。
2. Wait 通常被用于线程间交互，sleep 通常被用于暂停执行。
3. sleep(milliseconds)可以用时间指定来使他自动醒过来,如果时间不到你只能调用interreput()来强行打断;wait()可以用notify()直接唤起
4. sleep 是 Thread 类的静态方法。sleep的作用是让线程休眠制定的时间，在时间到达时恢复 
5. sleep必须捕获异常，而wait，notify和notifyAll不需要捕获异常

为什么是静态的, 可能是线程的调度由 CPU 来管, 一个线程不能对另外一个线程发号施令, 那 signal 又算什么呢?

### 4）用Java实现阻塞队列

这是一个相对艰难的多线程面试问题，它能达到很多的目的。第一，它可以检测侯选者是否能实际的用Java线程写程序；

第二，可以检测侯选者对并发场景的理解，并且你可以根据这个问很多问题。如果他用wait()和notify()方法来实现阻塞队列，你可以要求他用最新的Java 5中的并发类来再写一次。

Blocking Queue 的实现之, ArrayBlockingQueue

```java
ArrayBlockingQueue
    final Object[] items;
    int takeIndex;
    int putIndex;
    int count;
    final ReentrantLock lock;
    private final Condition notEmpty;
    private final Condition notFull;
    
    final E itemAt(int i) {
            return (E) items[i];
    }
    
    * Call only when holding lock.
    private void enqueue(E x) {
        final Object[] items = this.items;
        items[putIndex] = x;
        
        if (++putIndex == items.length)
            putIndex = 0; // 下次 enqueue 怎么办?
        
        count++;
        notEmpty.signal();
    }
    
    private E dequeue() {
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
    
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
                throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

常用操作, put, offer, poll, take

根据 count 来判断队列是否满了, 这种判断非常简便, enqueue 就不需要再做判断了

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

public void put(E e) throws InterruptedException {
    
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 和 lock 有什么区别
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

public boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException {    
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos); // waitNano
        }
        enqueue(e);
        return true;
        } finally {
            lock.unlock();
        }
    }
```

取元素

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}

public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}
```

因为每次取出元素后, 原始的位置都会被置为 null, 所以可以直接返回 itemAt (takeIndex)

```java
public int size() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return count;
    } finally {
        lock.unlock();
    }
}
```

### 6）用Java编程一个会导致死锁的程序，你将怎么解决？
    
这是我最喜欢的Java线程面试问题，因为即使死锁问题在写多线程并发程序时非常普遍，但是很多侯选者并不能写deadlock free code（无死锁代码？），
他们很挣扎。只要告诉他们，你有N个资源和N个线程，并且你需要所有的资源来完成一个操作。为了简单这里的n可以替换为2，越大的数据会使问题看起来更复杂。
通过避免Java中的死锁来得到关于死锁的更多信息。

转账问题?

```java
class Account {
  double balance;
  int id;

  void withdraw(double amount){
     balance -= amount;
  } 

  void deposit(double amount){
     balance += amount;
  } 

   void transfer(Account from, Account to, double amount){
        sync(from);
        sync(to);
           from.withdraw(amount);
           to.deposit(amount);
        release(to);
        release(from);
    }
}
```
Obviously, should there be two threads which attempt to run transfer(a, b) and transfer(b, a) at the same time, then a deadlock is going to occur because they try to acquire the resources in reverse order.

Try lock 解决死锁问题

```java
public class TestDeadLock4 implements Runnable{
    private boolean      flag;
    static ReentrantLock lock1 = new ReentrantLock();
    static ReentrantLock lock2 = new ReentrantLock();
    //统计发生死锁的次数
    private static int count;

    public TestDeadLock4(boolean flag) {
        this.flag = flag;
    }

    @Override
    public void run() {
        if(flag){
            while (true) {
                if(lock1.tryLock()){
                    System.out.println(flag+"线程获得了lock1");
                    try {
                        TimeUnit.MILLISECONDS.sleep(1);
                        try {
                            if(lock2.tryLock()){
                                System.out.println(flag+"获得了lock2");
                            }
                        } finally {
                            //同时获得Lock1和lock2，没有发生死锁，任务完成，退出循环
                            if(lock1.isHeldByCurrentThread()&&lock2.isHeldByCurrentThread()){
                                System.out.println(flag+"线程执行完毕"+"---------------------");
                                lock1.unlock();
                                lock2.unlock();
                                break;
                            }else{
                                //说明发生了死锁，只需要释放lock1
                                //统计变量也要注意多线程问题
                                synchronized (TestDeadLock4.class) {
                                    count++;
                                    System.out.println("发生了"+count+"次死锁");
                                }
                                lock1.unlock();
                            }
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }else{
            while (true) {
                if(lock2.tryLock()){
                    System.out.println(flag+"线程获得了lock2");
                    try {
                        TimeUnit.MILLISECONDS.sleep(1);
                        try {
                            if(lock1.tryLock()){
                                System.out.println(flag+"线程获得了lock1");
                            }
                        } finally {
                            if(lock1.isHeldByCurrentThread()&&lock2.isHeldByCurrentThread()){
                                System.out.println(flag+"线程执行完毕"+"---------------------");
                                lock1.unlock();
                                lock2.unlock();
                                break;
                            }else{
                                synchronized (TestDeadLock4.class) {
                                    count++;
                                    System.out.println("发生了"+count+"次死锁");
                                }
                                lock2.unlock();
                            }
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(new TestDeadLock4(true));
        Thread thread2 = new Thread(new TestDeadLock4(false));
        thread1.start();
        thread2.start();
    }
}
```

[useful link](https://my.oschina.net/u/2529036/blog/625291)

解释了 lockInterruptly 的例子

如果 method1 和 method2 同时被调用, 那么这两个方法会死锁

### 9) 什么是竞争条件？你怎样发现和解决竞争？

**race condition:**

A race condition occurs when two or more threads can access shared data and they try to change 
it at the same time. Because the thread scheduling algorithm can swap between threads at any time, you don't 
know the order in which the threads will attempt to access the shared data. Therefore, the result of the change 
in data is dependent on the thread scheduling algorithm, i.e. both threads are "racing" to access/change the data.

**preventing Race Conditions**

To prevent race conditions from occurring you must make sure that the critical section is executed as an atomic instruction. 
That means that once a single thread is executing it, no other threads can execute it until the first thread has left the critical section.

Race conditions can be avoided by proper thread synchronization in critical sections. 
Thread synchronization can be achieved using a synchronized block of Java code. 
Thread synchronization can also be achieved using other synchronization constructs like 
locks or atomic variables like java.util.concurrent.atomic.AtomicInteger.

这是一道出现在多线程面试的高级阶段的问题。大多数的面试官会问最近你遇到的竞争条件，以及你是怎么解决的。有些时间他们会写简单的代码，
然后让你检测出代码的竞争条件。可以参考我之前发布的关于Java竞争条件的文章。在我看来这是最好的java线程面试问题之一，
它可以确切的检测候选者解决竞争条件的经验，or writing code which is free of data race or any other race condition。
关于这方面最好的书是《Concurrency practices in Java》。

### 10) 你将如何使用thread dump？你将如何分析Thread dump？
    
1. 在UNIX中你可以使用kill -3，然后thread dump将会打印日志，在windows中你可以使用”CTRL+Break”。非常简单和专业的线程面试问题，但是如果他问你怎样分析它，就会很棘手。

2. jstack -l JAVA_PID > jstack.out

添加 -XX:+HeapDumpOnOutOfMemoryError 用来生成 coredump 文件, 他描述了内存的分布问题

-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${DOMAIN_HOME}/logs/mps"

这个是 coreDump

### 11) 为什么我们调用start()方法时会执行run()方法，为什么我们不能直接调用run()方法？
    
这是另一个非常经典的java多线程面试问题。这也是我刚开始写线程程序时候的困惑。现在这个问题通常在电话面试或者是在初中级Java面试的第一轮被问到。
这个问题的回答应该是这样的，当你调用start()方法时你将创建新的线程，并且执行在run()方法里的代码。但是如果你直接调用run()方法，
它不会创建新的线程也不会执行调用线程的代码。阅读我之前写的《start与run方法的区别》这篇文章来获得更多信息。

Thread 的 Start 方法

```java
public synchronized void start() {
    if (threadStatus != 0) throw new IllegalThreadStateException();
    group.add(this);
    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
            it will be passed up the call stack */
        }}
```

Causes this thread to begin execution; the Java Virtual Machine calls the run method of this thread

The result is that two threads are running concurrently: the current thread (which returns from the call to the
start method) and the other thread (which executes its run method).

### 12) Java中你怎样唤醒一个阻塞的线程？
    
这是个关于线程和阻塞的棘手的问题，它有很多解决方法。如果线程遇到了 IO 阻塞，我并且不认为有一种方法可以中止线程。

如果线程因为调用wait()、sleep()、或者join()方法而导致的阻塞，你可以中断线程，并且通过抛出InterruptedException来唤醒它。

我之前写的《How to deal with blocking methods in java》有很多关于处理线程阻塞的信息。

### 13)在Java中CycliBarriar和CountdownLatch有什么区别？
    
这个线程问题主要用来检测你是否熟悉JDK5中的并发包。这两个的区别是CyclicBarrier可以重复使用已经通过的障碍，而CountdownLatch不能重复使用。

### 14) 什么是不可变对象，它对写并发应用有什么帮助？
    
另一个多线程经典面试问题，并不直接跟线程有关，但间接帮助很多。这个java面试问题可以变的非常棘手，如果他要求你写一个不可变对象，或者问你为什么String是不可变的。

### 15) 你在多线程环境中遇到的常见的问题是什么？你是怎么解决它的？
    
多线程和并发程序中常遇到的有Memory-interface、竞争条件、死锁、活锁和饥饿。问题是没有止境的，如果你弄错了，将很难发现和调试。
这是大多数基于面试的，而不是基于实际应用的Java线程问题。

### 补充的其它几个问题：
    
1) 在java中绿色线程和本地线程区别？

2) 线程与进程的区别？

3) 什么是多线程中的上下文切换？

4)死锁与活锁的区别，死锁与饥饿的区别？

5) Java中用到的线程调度算法是什么？

6) 在Java中什么是线程调度？

7) 在线程中你怎么处理不可捕捉异常？(什么是不可捕捉异常?)

8) 什么是线程组，为什么在 Java 中不推荐使用？

9) 为什么使用 Executor 框架比使用应用创建和管理线程好？

10) 在Java中 Executor 和 Executors 的区别？

11) 如何在Windows和Linux上查找哪个线程使用的CPU时间最长？


### 50 道线程题目

[link](http://www.importnew.com/12773.html)


### javarevisited 面试题合集

[link](http://www.importnew.com/17232.html)

### 40 个多线程题目

[link](http://www.importnew.com/18459.html)


