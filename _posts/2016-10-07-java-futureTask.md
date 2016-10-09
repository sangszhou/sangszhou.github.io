---
layout: post
title: java future task and threadpool
categories: [java]
keywords: java
---

![](/images/posts/javaconcurrent/futuretask.png)

### Example

```java
public class FutureTaskExample {
	public static void main(String[] args) {
		MyCallable callable1 = new MyCallable(1000);
		MyCallable callable2 = new MyCallable(2000);

		FutureTask<String> futureTask1 = new FutureTask<String>(callable1);
		FutureTask<String> futureTask2 = new FutureTask<String>(callable2);

		ExecutorService executor = Executors.newFixedThreadPool(2);
		
		executor.execute(futureTask1);
		executor.execute(futureTask2);
		
		while (true) {
			try {
				if(futureTask1.isDone() && futureTask2.isDone()){
					System.out.println("Done");
					//shut down executor service
					executor.shutdown();
					return;
				}
				
				if(!futureTask1.isDone()){
                    //wait indefinitely for future task to complete
                    System.out.println("FutureTask1 output="+futureTask1.get());
				}
				
				System.out.println("Waiting for FutureTask2 to complete");
				String s = futureTask2.get(200L, TimeUnit.MILLISECONDS);
				if(s !=null){
					System.out.println("FutureTask2 output="+s);
				}
			} catch (InterruptedException | ExecutionException e) {
				e.printStackTrace();
			}catch(TimeoutException e){
				//do nothing
			}}}}
```

### source code

```java
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     
public class FutureTask<V> implements RunnableFuture<V> {
    private volatile int state;
    private Callable<V> callable;
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters; // 等待的节点
    
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
    
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
    
    private int awaitDone(boolean timed, long nanos) throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }
    
            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                    return s;
                }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                         q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos); // 在这就 block 了
            }
            else
                LockSupport.park(this);
            }}
```

试图获取数据的线程就挂在 WaitNodes 上, 被 LockSupport.park 了, 对于那些限时等待的请求, 如果线程醒来以后
发现数据仍然不可用, nano 就返回非正值 (state), 在 report 里 resolve state 对应的返回值

如果是不限时等待, 那就直接被无限 park

```java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
}}

// 一个个的 unpark 这些节点
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }}
        done();
        callable = null;        // to reduce footprint
    }
```

线程池执行 futureTask 请求

```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}

ThreadPoolExecutor

public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
            c = ctl.get();
        }
        
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
            }
            else if (!addWorker(command, false))
                reject(command);
}
```

从 ThreadPoolExecutor 中, 也没看到什么特殊情况, 肯定是 FutureTask 的 run 方法与众不同

```java
    public void run() {
        // 不可以重新运行
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

### ThreadPool 的继承关系

Executor <- ExecutorService <- AbstractExecutorService <- ThreadPoolExecutor

其中 Executor 只能有 execute 方法

ExecutorService 基本上该有的方法都有了


```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());

public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);}

public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
```

### BlockingQueue 的实现

LinkedBlockingQueue, ArrayBlockingQueue, PriorityBlockingQueue, DelayQueue, SynchronousBlockingQueue

ArrayBlockingQueue 和 DelayQueue 的源码都仔细的读过, LinkedBlockingQueue 应该也不成问题

PriorityBlockingQueue 的实现则复杂了些, 它没有使用已有的数据结构, 而是手动的写了 sink, siftUp 等函数

**SynchronousBlockingQueue**

Q: What is the difference? When I should use SynchronousQueue against LinkedBlockingQueue with capacity 1?

A:

the SynchronousQueue is more of a handoff, whereas the LinkedBlockingQueue just allows a single element. 
The difference being that the put() call to a SynchronousQueue will not return until there is a 
corresponding take() call, but with a LinkedBlockingQueue of size 1, the put() call (to an empty queue) will return immediately.

I can't say that i have ever used the SynchronousQueue directly myself, but it is the default BlockingQueue used for 
the Executors.newCachedThreadPool() methods. It's essentially the BlockingQueue implementation for when you don't 
really want a queue (you don't want to maintain any pending data).

Sync.Q. requires to have waiter(s) for offer to succeed. LBQ will keep the item and offer will finish immediately even if there is no waiter.

SyncQ is useful for tasks handoff. Imagine you have a list w/ pending task and 3 threads available waiting on the queue, try offer() with 1/4 of the list if not accepted the thread can run the task on its own. [the last 1/4 should be handled by the current thread, if you wonder why 1/4 and not 1/3]

Think of trying to hand the task to a worker, if none is available you have an option to execute the task on your own (or throw an exception). On the contrary w/ LBQ, leaving the task in the queue doesn't guarantee any execution.

Note: the case w/ consumers and publishers is the same, i.e. the publisher may block and wait for consumers but after offer or poll returns, it ensures the task/element is to be handled.

没怎么看明白
