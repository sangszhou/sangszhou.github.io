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
