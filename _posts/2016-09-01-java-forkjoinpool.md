---
layout: post
title:  "(Draft) Java ForkJoinPool"
date:   "2016-09-01 00:00:00"
categories: Java
keywords: Java, Concurrent
---

## ForkJoinPool demo

```java
class DefaultTaskScheduler
  static ForkJoinTask schedule(Function<Void, T> func)
    RecursiveTask task = new RecursiveTask() {
      public T compute { func.apply(null) }
    }

    ForkJoinPoolIns.execute(task)

    return task

static long parSort(List<Integer> nums, int left, int right, int threshold)
  if(right - left <= threshold) return ...

  int mid = (right - left) / 2 + left

  ForkJoinTask<Long> leftTask = DefaultTaskScheduler.schedule(Void -> parSort(nums, left, mid, threshold))
  ForkJoinTask<Long> rightTask = DefaultTaskScheduler.schedule(Void -> parSort(nums, mid, right, threshold))

  long leftInversion = leftTask.join
  long rightInversion = rightTask.join
```

上面是一段 java8 的代码，但是这里要分析的是 java7 的 ForkJoinPool 实现

## execute 流程

```java
void execute(ForkJoinTask task)
  if(task == null) throw new Exception
  forkOrSubmit(task)

void forkOrSubmit(ForkJoinTask task)
  ForkJoinWorkerThread w
  Thread t = Thread.currentThread
  if((t instanceof ForkJoinWorkerThread) && (w = t.pool == this))
    w.pushTask(task)
  else addSubmission(task)
```

和 线程池的一般套路相同，执行的过程只是把任务放到任务队列中，而不是马上执行，所以 execute 也没有返回值

当前线程是 ForkJoinWorkerThread 时，任务被放到自己的任务队列中，否则放到线程池的任务队列中

```java
ForkJoinWorkerThread:
void pushTask(ForkJoinTask t)
  ForkJoinTask[] q; int s, m
  if((q = queue) != null)
    long u = queueTop & q.length - 1 // u is index
    UNSAFE.putOrderedObject(q, u, t) // 放到队列的前面
    queueTop = s + 1
    if(s -= queueBase <= 2) pool.signalWork
    else growQueue
```

queueBase 的定义，Index of least valid queue slot, which is always the next position to steal from if nonempty. 其他线程可见，所以有 volatile 修饰。

queueTop 标识下一个被 push 或者 pop 的位置，这个值只会被当前线程修改，因此没有加 volatile 修饰

ForkJoinTask [] queue 数组的大小必须是 2 的幂，方便取模和移位运算

干完活的线程与其等着，不如去帮其他线程干活，于是他就去其他线程队列中窃取一个任务来执行，而在这是他们会访问同一队列，所以为了减少窃取任务线程和被窃取线程之间的竞争，通常会用到双端队列，被窃取任务线程用于从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行

```java
// 一堆位操作
void signalWork
  while there is too few total workers OR there is at least one waiter AND there are too few active workers OR the pool is terminating
    ...
    addWorker()

void addWorker
  Throwable ex = null
  ForkJoinWorkerThread t = null
  try
    t = factory.newThread(this)
    if(t == null) // loop
    else t.start
```

看看 ForkJoinWorkerThread 的 run 实现

```java
void run
  Throwable Exception = null
  try
    onStart()
    pool.work(this)
  finally
    onTermination(exception)

void work(ForkJoinWorkerThread w)
  boolean swept = false
  long c
  while(!w.termiate && ctl >= 0)
    if(!swept && （c << AC_SHIFT) <= 0)
      swept = scan(w, a)
    else if(tryAwaitWork(w, c))
      swept = false

scan(ForkJoinWorkerThread w, int a)
  if(b = queueBase != queueTop && q = submissionQueue != null && i = (q.length -1 & b) >= 0)
    long u = i << ASHIFT + ABASE
    if t = q[i] != null && queueBase == b && UNSAFE.compareAndSwapObject(q, u, t, null)
      queueBase = b + 1
      w.execTask(t)

void execTask(ForkJoinTask t)
  currentSteal = t
  for( ; ; )
    if(t != null) t.doExec
    if(queueTop == queueBase) break
    // locallyFifo 需要
    t = locallyFifo ? locallyDeqTask : popTask

  ++ stealCount
  currentSteal = null

ForkJoinTask locallyDeqTask
  ForkJoinTask t
  ForkJoinTask[] q = queue
  while(queueTop != queueBase)
    if(t = q[i = m&b] != null && queueBase == b && UNSAFE.compareAndSwapObject(q, i << ASHIFT + ABASE, t, null))
      queueBase = b + 1
      return t // 拿到了 queueBase，从自己的队列中取一个元素

ForkJoinTask popTask
  ForkJoinTask [] q = queue
  if(q != null && (m = q.length -1) >= 0)
    for(int s; (s = queueTop) != queueBase; )
      int i = m & --s
      ForkJoinTask t = q[i]
      if(t == null) break
      if(UNSAFE.compareAndSwapObject(q, u, t, null))
        queueTop = s
        return t    
```
popTask 是从 queue 的取来的，locallyDeqTask 是从 queueBase 出来了，popTask 是从 queueTop 算的

```java
ForkJoinTask:
void doExec
  if(status >= 0) // accessed directly by pool and workers
    boolean completed
    try
      completed = exec
    if(completed) setCompletion(NORMAL)


boolean exec()
  result = compute
  return true
```
执行任务的过程就是我们自己定义的 compute 的过程

## Join 过程

```java
V join
  if(doJoin() != NORMAL) return reportResult()
  else getRawResult

int doJoin()
    Thread t; ForkJoinWorkerThread w; int s; boolean completed;

    if(( w = t = Thread.currentThread) instanceof ForkJoinWorkerThread)
      if(s = status < 0)
        return s
      if(t.unpushTask(this))
        completed = exec
        if(completed) return setCompletion(NORMAL)
      return w.joinTask(this) //
    else return externalAwaitDone // 外部

ForkJoinWorkerThread:
int joinTask(ForkJoinTask joinMe)
  ForkJoinTask prevJoin = currentJoin
  currentJoin = joinMe
  for(int s, retries = MAX_HELP; ;)
    if(s = joinMe.status < 0)
      currentJoin = prevJoin
      return s
    if(retries > 0)
      if(queueTop != queueBase)
        if(! localHelpJoinTask(joinMe))
          retries = 0
        else if (retries == MAX_HELP >>> 1)
          -- retries
          if(tryDeqAndExec(joinMe) >= 0)
            Thread.yield
        else
          retries = helpJoinTask(joinMe) ? MAX_HELP : retries - 1
    else
      retries = MAX_HELP
      pool.tryAwaitJoin(joinMe)
```

```java
// specialized version of popTask to pop only if topmost element
// is the given task. Called only by this thread
boolean unpushTask(ForkJoinTask t)
  ForkJoinTask [] q
  int s
  if(q = queue != null && s = queueTop != queueBase &&
    UNSAFE.compareAndSwapObject(q, xxx, t, null))
    queueTop = s
    return true
  return false
```


```java
void tryAwaitJoin(ForkJoinTask joinMe)
  int s
  Thread.interrupted
  if(joinMe.status >= 0)
    if(tryPreBlock)
      joinMe.tryAwaitDone(0L)
      postBlock

void tryAwaitDone(long millis)
  if(s == status > 0 || s == 0 && UNSAFE.compareAndSwapInt(this, statusOffset, 0, SIGNAL) && status > 0)
    synchronized(this)
      if(status > 0) wait(millis)


int externalAwaitDone
  if s = status >= 0
    synchronized(this)
      while(s = status >= 0)
        if(s = 0) UNSAFE.compareAndSwapInt(this, statusOffset, 0, SIGNAL)
        else
          try wait
```
