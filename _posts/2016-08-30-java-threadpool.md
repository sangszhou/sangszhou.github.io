---
layout: post
title:  "Java ThreadPoolExecutor 代码分析"
date:   "2016-08-29 17:50:00"
categories: Java
keywords: Java, Concurrent
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
    reExecutePeridic
```

periodic 和 非 periodic 的区别是执行 run 还是 runAndReset, 对于单次任务，执行完毕后就结束了，对于循环逻辑，此次执行完毕后再把自己放回到任务队列中。因为 ScheduledFutureTask 是 FutureTask 的子类，一旦 runnable 执行完毕，state 就会发生变化，不能被执行两次。为了避免再次创建 FutureTask 的开销，就直接重用自己，这也是 runAndReset 的意义，不调用 set(Result) 所以 State 也不会变。此外，因为循环调用的方法都是没有返回值的，所以也没有 set result 的必要。

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

注意 Queue 并没有使用 DelayQueue, 而是自己的实现，不知道为什么。
