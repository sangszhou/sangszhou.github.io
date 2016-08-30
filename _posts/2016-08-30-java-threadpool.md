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

execute 分为三种情况
- If fewer than corePoolSize threads are running, try to start a new thread with th given command as its first task.
- If a task can be successfully queued, then we still need to double check whether we should have added a thread.
- If we cannot queue task, then we try to add a new thread.

execute 和 addworker 应该放到一起看，因为 execute 对 addWorker 返回状态的含义有假设，比如当 addWorker 返回成功时，线程池的状态肯定是好的，当 addWorker 失败时，要么 worker 超过了 capacity, 要么线程池状态不对，这两种结果下都要 reject command

`addWorker` 是核心函数

```scala
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

```
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
				afterExecution(task, exception)		finally
				w.unlock
```
getTask 是从 workQueue 中获取数据，如果拿不到就返回 null 返回 null 时要么超时，要么 threadpool 状态异常

```
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

```
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

```
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

```
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

```
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

```
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
