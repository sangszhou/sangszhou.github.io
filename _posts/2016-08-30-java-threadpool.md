---
layout: post
title:  "Java ThreadPoolExecutor 代码分析"
date:   "2016-08-29 17:50:00"
categories: java
keywords: java, concurrent
---

### 正常逻辑，处理任务并退出

```java
public void execute(Runnable command)
	if(workerCount < corePoolSize) // 总是创建 worker
		if(addWorker(command, true))
			return;
	if(isRunning && workQueue.offer(command))
		// recheck
		if(!isRunning && remove(command))
			reject(command) // threadpool is shutdown now
		if(workerCount == 0)
			addWorker(null, false)
	else if (!addWorker(command, false)) // workQueue reject command
		reject(command)
```

execute 分为三种情况:

1. If fewer than corePoolSize threads are running, try to start a new thread with th given command as its first task.
2. If a task can be successfully queued, then we still need to double check whether we should have added a thread.
3. If we cannot queue task, then we try to add a new thread.

execute 和 addworker 应该放到一起看，因为 execute 知道 addWorker 返回状态的含义，比如当 addWorker 返回成功时，线程池的状态肯定是好的，当 addWorker 失败时，要么 worker 超过了 capacity, 要么线程池状态不对，这两种结果下都要 reject command

`addWorker` 是核心函数, 它在线程池中添加一个新的 worker, 线程的创建 delegate 给 threadFactory, 当创建的线程返回 null 时，说明系统出错, 此时返回 false.

```java
boolean addWorker(Runnable firstTask, boolean core)
	retry:
	for(;;)
		if(state >= SHUTDOWN && !(state ==SHUTDOWN && firstTask == null && !workQueue.isEmpety))
			return false; // 当 stop 或者 shutdown 但没任务执行时，return false
		for(;;)
			int wc = workerCount
			if(wc >= CAPACITY || wc >= (core ? corePoolSize : maxPoolSize))
				return false
			if(compareAndIncrementWorkerCount))
				break retry; // 拿到了可以创建 worker 的许可
			c = ctl.get
			if(state != runStateOf(c)) continue retry; // 状态发生了变化，重试

	Worker w = new Worker(firstTask)
	Thread t = w.thread;
	mainLock.lock //lock threadpool before adding worker to queue
	try
		if(t == null || (ready to shutdown))
			decrementWorkerCount
			tryTerminate
			return false

		workers.add(w)
	finally
		mainLock.unlock

	t.start
	return true
```

ready to shutdown 是靠 state == shutdown && (state == shutdown && firstTask == null && !workerQueue.isEmpty) 决定的

```java
Worker extends AQS with Runnable //AQS 可能是为了显示当前 worker 的状态
Worker.run() { runWorker(this) }

void runWorker(Worker w)
	Runnable task = w.firstTask
	while(task != null || (task = getTask() != null))
		w.lock // 锁住 worker
		try
			beforeExecution(w.thread, task) // 为了扩展
			try
				task.run
			finally
				afterExecution(task, exception)		
		finally
			w.unlock
```
getTask 是从 workQueue 中获取数据，如果拿不到就返回 null 返回 null 时要么超时，要么 threadpool 状态异常

```java
Runnable getTask
	boolean timeout = false; // Did last pool timeout?
	retry:
	for(;;)
		if(state >= STOP && workQueue.isEmpty)
		// 减少一个 worker，但不一定减少自己
			decrementWorkerCount
			return null;
		for(;;)
			boolean timed = allowCoreThreadTimeout || wc > corePoolSize
			if(wc <= maximumPoolSize && (!timeout && timed))
				break // 尝试获取 task
			if(comprareAndDecrementWorkerCount)
				return null; // 第一次 loop 不会走到这里
			if(state changed) continue retry

		Runnable r = timed ? workerQueue.pool(keepAliveTime) : workQueue.take
		if(r != null) return r
		else timeout = true
```
如果上一次获取 worker 超时，就会减少一个 worker, 但被减少的却不一定是自己，这个时候 getTask 知道调用自己的函数会对自己返回 null 时的动作，processWorkExit

处理 Shutdown, Exception 等逻辑

```java
void processWorkerExit(Worker w, boolean completedAbruptly)
	if(completedAbruptly) decrementWorkerCount
	mainLock.lock
	workers.remove(w)
	mainLock.unlock

	tryTerminate
	if(state < stop)
		if(!completedAbtruptly)
			int min = allowCoreThreadTimeout ? 0 : corePoolSize
			if(min == 0 && !workQueue.isEmpty())
				min = 1
			if(workerCount >= min) return;
		addWorker(null, false) // 还要再创建 worker
```
处理 worker 退出, 当 worker 在 runWorker 被中强行中断时，completedAbruptly 变量被置为 false, 此时 processWorkerExit 是会添加一个新的 worker 到线程池，如果这个线程正常结束，那么判断线程池是否还剩下一个线程，如果完全没有线程的话就再创建一个。其实我觉得倒是可以设置一个特殊的线程，使得线程池总是至少有一个线程，这样使代码能够缺少很多判断逻辑。

### 退出

退出有两种，第一种是 shutdown 第二种是 shutdownRightNow

```java
void shutdown()
	mainLock.lock
	try
		checkShutdownStatus
		advanceRunState(SHUTDOWN)
		interruptIdleWorkers()
		onShutdown()
	finally
		mainLock.unlock
	tryTerminate

```
checkShutdownState 是用来检查 caller 是否有 shutdown threadpool 的权限。
advanceRunState(SHUTDOWN) 是一个循环 CAS，置 state 为 shutdown
interruptIdleWorkers 遍历所有的 worker 把那些不在 lock 状态的线程 interrupt
onShutdown 肯定就是留作扩展的函数的了

```java
List<Runnable> shutdownNow
	mainLock.lock
	try
		checkShutdownAccess
		advanceRunState(STOP)
		interruptWorkers()
		tasks = drainQueue
	finally
		mainLock.unLock
	tryTerminate
	return tasks

List<Runnable> drainQueue
	BlockingQueue<Runnable> q = workerQueue
	List<Runnable> taskList = new ArrayList
	q.drainTo(taskList)
	if(!q.isEmpty)
		for(Runnable r: q.toArray(new Runnable[0]))
			if(q.remove(r))
				taskList.add(r)
	return taskList
```
当 Queue 是 delayQueue 或者调用 poll, draintTo 会失败的 queue 时，那就需要一个个的删除元素了

tryTerminate 在很多函数中被调用
transition to TERMINATED state if either (SHUTDOWN and pool and queue is empty) or (STOP and pool empty).  This method must be called following any action that might make termination possible -- reducing worker count or removing taks from the queue during shutdown.

```java
void tryTerminate
	for( ; ;)
		if(isRunning || runStateAtLeast(TIDYING) || runState == SHUTDOWN && !workerQueue.isEmpty())
			return
		if(workerCount != 0) //eligibe to terminate
			interruptIdleWorker(ONLY_ONE)
			return
		mainLock.lock
		try
			if(ctl.compareAndSet(c, ctlOf(TIDYING, 0)))
				try
					terminated
				finally
					ctl.set(TERMINATED)
					termination.signalAll
		finally
			mainLock.unlock
```
terminated 用于扩展，供子类继承。
workerCount != 0 时调用 interruptIdleWorker 是什么意思？
termination.signalAll, 什么时候线程被挂在这个 condition 上的？


`purge`
Tries to remove from the worker queue all tasks that have been cancelled.

```java
void purge
	BlockingQueue<Runnable> q = workQueue
	Iterator<Runnable> it = q.iterator
	while(it.next)
		Runnable r = it.next
		if(r instanceof Future && r.isCancell)
			it.remove
	catch ConcurrentModificationException fallThrough
		for(object r: q.toArray)
			if(r instanceof Future && r.isCancelled)
				q.remove(r)
	tryTerminate
```

## ScheduledThreadPool

### Schedule 流程

scheduleAtFixedRate 的要点就是什么时候把下一个要执行的任务放到队列中 

BlockingQueue<Runnable> workQueue // ThreadPool 中的 queue, 默认使用 DelayedWorkQueue, 这个 Queue 很像 DelayedQueue 
但是并不是, 其内部的实现是 heap, 每次插入以后都会对 Heap 重组一下 siftUp 

```java
static class DelayedWorkQueue extends AbstractQueue<Runnable> implements BlockingQueue<Runnable>
        private RunnableScheduledFuture<?>[] queue =
            new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
```

```java
ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initiaDelay, long period, TimeUnit unit)
  ScheduledFutureTask<Void> sft = new ScheduledFutureTask(command, null, triggerTime(initiaDelay, unit), NANOSECONDS)

  RunnableScheduledFuture<Void> t = decoreTask(command, sft)

  // outtask is the actual taks to be re-enqueued by reExecutePeridic
  stf.outerTask = t
  delayedExecute(t)
  return t;


RunnableScheduledFuture<V> decoreTask(Runnable runnable, RunnableScheduledFuture task)
  return task; // 为扩展

void delayedExecute(RunnableScheduledFuture task)
  if(isShutdown) reject(task)
  else
    getQueue.add(task)
    // 第二个
    if(isShutdown && !canRunInCurrentState(task.isPeriodic) && remove(task))
      task.cancel
    else
      ensurePrestart()

void ensurePrestart
  int wc = workerCount(ctl.get)
  if(wc < corePoolSize)
    addWorker(null, true)
  else if(wc == 0)
    addWorker(null, false)
```

scheduleAtFixedRate 函数只是把要执行的 task 放到等待队列中，然后保证线程池中有可用线程。当线程池已经被关闭时，reject task.
因为这是 scheduled threadpool, 可以想象 pool 里的线程肯定都在等着可用的 task 出现在队列的前面，然后运行 Task

实际上，代码的执行逻辑就是这样。queue 中的 task 都继承 Runnable 接口，线程拿到 task 后执行其 run 函数。定时任务都是 ScheduledFutureTask, 他们的 run 函数定义如下

```java
void run
  boolean periodic isPeriodic
  if(!canRunInCurrentState(periodic))
    cancel(false)
  else if(!periodic)
    ScheduledFutureTask.super.run
  else if(ScheduledFutureTask.super.runAndRest)
    setNextRunTime
    reExecutePeriodic
```

periodic 和 非 periodic 的区别是执行 run 还是 runAndReset, 对于单次任务，执行完毕后就结束了，对于循环逻辑，此次执行完毕后再把自己放回到任务队列中。因为 ScheduledFutureTask 是 FutureTask 的子类，
一旦 runnable 执行完毕，state 就会发生变化，此变化是不可逆的, 不能被执行两次。为了避免再次创建 FutureTask 的开销，就直接重用自己，这也是 runAndReset 的意义，不调用 set(Result) 所以 State 也不会变。
此外，因为循环调用的方法都是没有返回值的，所以也没有 set result 的必要。

FutureTask 的 run 方法会先检查自己的状态, 完成后修改状态, runAndSet 完成后不修改状态。至于为什么用 FutureTask 我想是重用吧, 从 runAndSet 的注释上也可以看出来。

```java
void ScheduledFutureTask.setNextRunTime
  long p = period
  if(p > 0) time += p
  else time = triggerTime(-p)

void reExecutePeridic(RunnableScheduledFuture task)
  if(canRunInCurrentState(true))
    super.getQueue.add(task)
    if(!canRunInCurrentState(true) && remove(task))
      task.cancel(false)
    else ensurePrestart
```

ScheduledFutureTask 是 ScheduledThreadPool 的内部类，在ScheduledFutureTask 中调用 ScheduledThreadPool 的方法也不需要指明实例变量。为了区分函数的所属类，显示标注了 setNextRunTime 是 ScheduledFutureTask 的函数。

<!--注意 Queue 并没有使用 DelayQueue, 而是自己的实现，不知道为什么。-->

## Threadpools

ExecutorService 的生命周期包括三种状态：运行、关闭、终止。创建后便进入运行状态，当调用了 shutdown（）方法时，
便进入关闭状态，此时意味着 ExecutorService 不再接受新的任务，但它还在执行已经提交了的任务，当素有已经提交了的任务执行完后，
便到达终止状态。如果不调用 shutdown（）方法，ExecutorService 会一直处在运行状态，不断接收新的任务，执行新的任务，
服务器端一般不需要关闭它，保持一直运行即可。

```
public static ExecutorService newFixedThreadPool(int nThreads)
创建固定数目线程的线程池。

public static ExecutorService newCachedThreadPool()
创建一个可缓存的线程池，调用execute将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线 程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。

public static ExecutorService newSingleThreadExecutor()
创建一个单线程化的Executor。

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
创建一个支持定时及周期性的任务执行的线程池，多数情况下可用来替代Timer类。

public static ScheduledExecutorService newSingleThreadScheduledExecutor()
```

**newCachedThreadPool()**

* 缓存型池子，先查看池中有没有以前建立的线程，如果有，就 reuse 如果没有，就建一个新的线程加入池中
* 缓存型池子通常用于执行一些生存期很短的异步型任务 因此在一些面向连接的 daemon 型 SERVER 中用得不多。
  但对于生存期短的异步任务，它是 Executor 的首选。
* 能 reuse 的线程，必须是 timeout IDLE 内的池中线程，缺省 timeout 是 60s,超过这个 IDLE 时长，线程实例将被终止及移出池。
  
**newFixedThreadPool(int)**

* newFixedThreadPool 与 cacheThreadPool 差不多，也是能 reuse 就用，但不能随时建新的线程
* 其独特之处:任意时间点，最多只能有固定数目的活动线程存在，此时如果有新的线程要建立，只能放在另外的队列中等待，直到当前的线程中某个线程终止直接被移出池子。
* 和 cacheThreadPool 不同，FixedThreadPool 没有 IDLE 机制（可能也有，但既然文档没提，肯定非常长，类似依赖上层的 TCP 或 UDP IDLE 机制之类的），所以 FixedThreadPool 多数针对一些很稳定很固定的正规并发线程，多用于服务器。
* 从方法的源代码看，cache池和fixed 池调用的是同一个底层池，只不过参数不同:
    fixed 池线程数固定，并且是0秒IDLE (无IDLE)
    cache 池线程数支持 0-Integer.MAX_VALUE(显然完全没考虑主机的资源承受能力），60 秒 IDLE
* newScheduledThreadPool(int)
    newScheduledThreadPool(int)
    这个池子里的线程可以按 schedule 依次 delay 执行，或周期执行
* SingleThreadExecutor
    单例线程，任意时间池中只能有一个线程
    用的是和 cache 池和 fixed 池相同的底层池，但线程数目是 1-1,0 秒 IDLE (无 IDLE)

一般来说，CachedTheadPool 在程序执行过程中通常会创建与所需数量相同的线程，然后在它回收旧线程时停止创建新线程，
因此它是合理的 Executor 的首选，只有当这种方式会引发问题时（比如需要大量长时间面向连接的线程时），才需要
考虑用 FixedThreadPool。（该段话摘自《Thinking in Java》第四版）

```java
public ThreadPoolExecutor (int corePoolSize, int maximumPoolSize, long keepAliveTime, 
    TimeUnit unit,BlockingQueue<Runnable> workQueue)
```

* corePoolSize：线程池中所保存的核心线程数，包括空闲线程

* maximumPoolSize：池中允许的最大线程数

* keepAliveTime：线程池中的空闲线程所能持续的最长时间

* unit：持续时间的单位

* workQueue：任务执行前保存任务的队列，仅保存由 execute 方法提交的 Runnable 任务

根据 ThreadPoolExecutor 源码前面大段的注释，我们可以看出，当试图通过 excute 方法讲一个 Runnable 任务
添加到线程池中时，按照如下顺序来处理：

1. 如果线程池中的线程数量少于 corePoolSize，即使线程池中有空闲线程，也会创建一个新的线程来执行新添加的任务；
2. 如果线程池中的线程数量大于等于 corePoolSize，但缓冲队列 workQueue 未满，
   则将新添加的任务放到 workQueue 中，按照 FIFO 的原则依次等待执行（线程池中有线程空闲出来后依次将缓冲队列中的任务交付给空闲的线程执行）；如果线程池中的线程数量大于等于 corePoolSize，且缓冲队列 workQueue 已满，但线程池中的线程数量小于 maximumPoolSize，则会创建新的线程来处理被添加的任务；
3. 如果线程池中的线程数量大于等于 corePoolSize，且缓冲队列 workQueue 已满，但线程池中的线程数量小于 maximumPoolSize，则会创建新的线程来处理被添加的任务
4. 如果线程池中的线程数量等于了 maximumPoolSize，有 4 种才处理方式（该构造方法调用了含有 5 个参数的构造方法，并将最后一个构造方法为 RejectedExecutionHandler 类型，它在处理线程溢出时有 4 种方式，这里不再细说，要了解的，自己可以阅读下源码）。

总结起来，也即是说，当有新的任务要处理时，先看线程池中的线程数量是否大于 corePoolSize，再看缓冲队列 workQueue 是否满，最后看线程池中的线程数量是否大于 maximumPoolSize。

另外，当线程池中的线程数量大于 corePoolSize 时，如果里面有线程的空闲时间超过了 keepAliveTime，就将其移除线程池，这样，可以动态地调整线程池中线程的数量。

```java
public static ExecutorService newCachedThreadPool() {  
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,  
                                  60L, TimeUnit.SECONDS,  
                                  new SynchronousQueue<Runnable>());  
}
```

它将 corePoolSize 设定为 0，而将 maximumPoolSize 设定为了 Integer 的最大值，线程空闲超过 60 秒，将会从线程池中移除。
由于核心线程数为 0，因此每次添加任务，都会先从线程池中找空闲线程，
如果没有就会创建一个线程（SynchronousQueue决定的，后面会说）来执行新的任务，并将该线程加入到线程池中，
而最大允许的线程数为 Integer 的最大值，因此这个线程池理论上可以不断扩大。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {  
    return new ThreadPoolExecutor(nThreads, nThreads,  
                                  0L, TimeUnit.MILLISECONDS,  
                                  new LinkedBlockingQueue<Runnable>());  
}
```

它将 corePoolSize 和 maximumPoolSize 都设定为了 nThreads，这样便实现了线程池的大小的固定，不会动态地扩大，
另外，keepAliveTime 设定为了 0，也就是说线程只要空闲下来，就会被移除线程池，敢于 LinkedBlockingQueue 下面会说

### 几种排序策略

**直接提交**

缓冲队列采用 SynchronousQueue，它将任务直接交给线程处理而不保持它们。如果不存在可用于立即运行任务的线程
(即线程池中的线程都在工作)，则试图把任务加入缓冲队列将会失败，因此会构造一个新的线程来处理新添加的任务，
并将其加入到线程池中。直接提交通常要求无界 maximumPoolSizes（Integer.MAX_VALUE） 以避免拒绝新提交的任务。
newCachedThreadPool 采用的便是这种策略

**无界队列**
使用无界队列（典型的便是采用预定义容量的 LinkedBlockingQueue，理论上是该缓冲队列可以对无限多的任务排队）将导
致在所有 corePoolSize 线程都工作的情况下将新任务加入到缓冲队列中。这样，创建的线程就不会超过 corePoolSize，也
因此，maximumPoolSize 的值也就无效了。当每个任务完全独立于其他任务，即任务执行互不影响时，适合于使用无界队列。
newFixedThreadPool采用的便是这种策略

**有界队列** 当使用有限的 maximumPoolSizes 时，有界队列（一般缓冲队列使用 ArrayBlockingQueue，并制定
队列的最大长度）有助于防止资源耗尽，但是可能较难调整和控制，队列大小和最大池大小需要相互折衷，需要设定合理的参数