---
layout: post
title:  "Java DelayQueue 分析"
date:   "2016-08-30 10:50:00"
categories: java
keywords: java, concurrent, blockingQueue
---

@todo update

1. poll 应该放到 offer 前面写
2. DelayedQueue 和 Blocking queue 一起写, 并总结子类的用途


### FAQ 

**新加入元素的的 delay 比老元素更小怎么办?**

这个时候在 offer 函数中, q.offer(e) == e 就成立了, wakeup waiting thread



DelayedQueue 中的元素类型为 `? extends Delayed`, 也就说 queue 内的元素在 getDelay 返回 0 时才有效，
在 getDelay > 0 时尝试获取元素的请求都将被阻塞或返回空

DelayQueue 的成员变量有两个，一个是 PriorityQueue, 另一个是 ReentrantLock. Lock 是用来保护 PriorityQueue 的入队出队，PriorityQueue 是用来保存元素的，getDelay 值小的元素放到队列的前面
Condition 用来挂载被锁住的线程，当元素有效时，会 signal 唤醒一个线程，整个 DelayQueue 的实现中都没有出现 signalAll

```java
transient ReentrantLock lock
PriorityQueue<E> q
Condition available;
```

## put, offer 流程

```java
public void put(E e) {
    offer(e);
}

boolean offer(E e)
  final ReentrantLock lock = this.lock
  lock.lock()
  try    
    q.offer(e)
    if(q.peek() == e) //
      leader = null
      available.signal()
      return true
  finally
    lock.unlock()
```

leader 也是 Queue 的成员变量，当 queue 非空时，但元素不可用时，多个线程调用 poll 或者 take ，第一个调用的线程被设置为 leader, 它被设置有限的等待时间，当它被唤醒时，肯定有一个元素可用了，其他的线程都被无期限的挂在 available Condition 上。按照注释的说明，这样做的好处是 "to minimize unnecessary timed waiting". 这是因为每次队列元素可用只能供一次 take 函数的调用，大部分醒来后根本拿不到数据，那么干脆就是拿到数据的线程唤醒下一个线程。

从上面的代码来看，当被添加的元素成为队列的首个元素时，有两种可能的原因，第一种是队列为空，第二种是 getDelay 函数返回了更小的等待时间。无论是哪种情况，都要求 leader 被唤醒重设等待时间。假如 leader 本身就为空，表示当前没有等待
的线程，那么 available.signal 只是一个空操作，假如有的话，就唤醒 leader 然他重设等待时间。？signal 是按照顺序来的么？

## take poll 流程

```java
E take() throw InterruptedException
  ReentrantLock lock = this.lock
  lock.lockInterruptly
  try
    for(;;)
      E first = q.peek
      if(first == null)
        available.await // 无限期等待，且不设置 leader
      else
        long delayedTime = first.delay(NANOSECONDS)
        if(delayedTime <= 0)
          return q.poll // 直接返回
        first = null;
        if(leader != null)
          available.await // 无限期等待，等待 leader 唤醒自己 (1)
        else
          Thread thread = Thread.currentThread
          leader = thread
          try
            leader.awaitNanos(delay)
          finally
            if(leader == thread) // 等醒来时
              leader = null
  finally
    if(leader == null && q.peek != null)
      available.signal // 唤醒一个无限期等待的线程 (2)
    lock.unlock
```

在 (2) 处，leader 会唤醒一个 follower, follower 在 (1) 处醒来，再次进入 for 循环，第一个被唤醒的 follow 会变成 leader

```java
E poll(long timeout, TimeUnit unit) throws InterruptedException
  long nanos = unit.toNanos(timeout)
  ReentrantLock lock = this.lock
// acquire this lock unless current thread is interrupted
  lock.lockInterruptly

  try
    for(;;)
      E first = q.peek
      if(first == null)
        if(nanos <= 0)
          return null
        else
          nanos = available.awaitNanos(nanos)
      else // first != null
        long delay = first.getDelay(NANOSECONDS)
        if(delay <= 0)
          return q.poll
        if(nanos <= 0)
          return null
        first = null
        if(nanos < delay || leader != null)
          nanos = available.awaitNanos(nanos) // 返回未睡眠够的时长
        else
          Thread thisThread = Thead.currentThread
          leader = thisThread
          try
            long timeLeft = available.awaitNanos(delay)
            nanos -= delay - timeLeft
          finally // finally after return? how to execute
            if(leader == thisThead)
              leader = null
  finally
    if(leader == null && q.peek() != null)
      available.signal
    lock.unlock
```

代码和 take 类似，比 take 稍微复杂些。具体表现在，当超时时长小于 delay 时，不要把自己设置为 Leader， 因为 leader 有义务唤醒 follower, 而超时时长小于 delay 说明自己离开时，还是没有可以用的数据(以目前的情况来说)。当自己(超时)返回时，还是会调用最后一个 finally, 如果此时还没有 leader, 会唤醒一个线程，对于那些设置了超时的 poll 线程，它们因为 nanos 还有剩余时间，仍然不会把自己设置成 leader，这个算法真是巧妙。


## drainTo 函数

```java
int drainTo(Collection<? super E> c)
  ReentrantLock lock = this.lock
  lock.lock
  try
    int n = 0
    for(E e; (e = peekExpired) != null; )
      c.add(e)
      q.poll
      ++n
    return n
  finally
    lock.unlock
```

peekExpired 是无视 getDelay 的 peek, 不知道为什么不直接用 q.peek

### BlockingQueue Implementations

```
ArrayBlockingQueue
DelayQueue
LinkedBlockingQueue
PriorityBlockingQueue
SynchronousQueue
```

```
	    Throws Exception	Special Value	Blocks	Times Out
Insert	add(o)	            offer(o)	    put(o)	offer(o, timeout, timeunit)
Remove	remove(o)	        poll()	        take()	poll(timeout, timeunit)
Examine	element()	        peek()	 	 

Throws Exception: 
If the attempted operation is not possible immediately, an exception is thrown.
Special Value: 
If the attempted operation is not possible immediately, a special value is returned (often true / false).
Blocks: 
If the attempted operation is not possible immedidately, the method call blocks until it is.
Times Out: 
If the attempted operation is not possible immedidately, the method call blocks until it is, but waits no longer than the given timeout. Returns a special value telling whether the operation succeeded or not (typically true / false).
```

