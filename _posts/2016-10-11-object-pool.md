在真正开始写代码前，我们先来梳理下一个对象池需要完成哪些功能。

1. 如果有可用的对象，对象池应当能返回给客户端
2. 客户端把对象放回池里后，可以对这些对象进行重用
3. 对象池能够创建新的对象来满足客户端不断增长的需求
4. 需要有一个正确关闭池的机制来确保关闭后不会发生内存泄露

```java
public interface Pool<T>
    T get();
    void release(T t);
    void shutdown();
```

现在来实现这个接口

```java
abstract class AbstractPool<T> implements Pool<T>
    public final void release(T t)
        if(isValid(t))
            returnToPool(t)
        else
            handleInvalidReturn(t)
    
    abstract void handleInvalidReturn(T t)
    abstract void returnToPool(T t)
    abstract boolean isValid(T t)
```

一个池子只是用来保存对象, 所以它不应该手段判断池内对象的有效性

```java
public static interface Validator<T>
    boolean isValid(T t);
    
    // cleanup activities
    void invalidate(T t);
```

现在, 对象池还不能创建对象, 我们可以搞一个 Factory 专门用于对象的创建

```java
public interface ObjectFactory<T>
    public abstract T createNew()
```

为了让我们的 Pool 能够在并发程序里使用, 我们创建一个阻塞的对象池, 当没有对象可用时, 
让客户端先阻塞住。

```java
public interface BlockingPool<T> extends Pool <T>
    T get()
    T get(long time, TimeUnit unit) throws InterruptedException
```

BoundedBlockingPool 的实现

```java
public final class BoundedBlockingPool extends AbstractPool implements BlockingPool
    private int size
    private BlockingQueue objects
    Validator validator
    ObjectFactory objectFactory
    ExecutorService executor = Executors.newCachedThreadPool();
    volatile boolean shutdownCalled;
    
    T get(long timeout, TimeUnit unit)
        if(!shutdownCalled)
            T t = null
            try 
                t = objects.poll(timeout, unit)
                return t
            catch(InterruptedException ie)
                THread.currentThread.interrupted
            return t
        throw new IllegalStateException()
    
    T get()
        if(!shutdownCalled)
            T t = null
            try 
                t = objects.take
            catch InterruptedException
        throw new IllegalStateException
    
    void shutdown
        shutdownCalled = true
        executor.shutdownNow
        clearResource
    
    private void clearResources
        for(T t: objects)
            validator.invalidate(t)
    
    void returnToPool(T t)
        if(validator.isValid(t)
            executor.submit(new ObjectReturner(objects, t))
    
    isValid(T t)
        return validator.isValid(t)
        
    private class ObjectReturner implements Callable
        BlockingQueue queue
        E e
        
        ObjectReturner(BlockingQueue queue, E e)
        
        Void call()
            while(true)
                queue.put(e)
                break
             return null
```

一个 JDBC 的例子

```java
public final class JDBCConnectionValidator implements Validator<Connection>
    boolean isValid(Connection)
        if(conn == null) true
        return !conn.isClosed();
    
    void invalidate(Connection con)
        con.close

public class JDBCCOnnectionFactory implements ObjectFactory<Connection>
    String ConnectionURL
    String username
    String password
    
    Connection createNew
        return DriverManager.getConnection(
            connectionURL, username, password)

public static void main(String[] args)
    Pool<Connection> pool = new BoundedBlockingPool<Connection>(10,
        new JDBCConnectionValidator(),
        new JDBCConnectionFactory)
    // do whatever you like
```

创建 Factory 的 Factory

```java
public final class PoolFactory
    private PoolFactory() {}
    
    public static <T> Pool<T> newBoundedBlockingPool(int size,
        ObjectFactory<T> factory,
        Validator<T> validator) {
        
            return new BoundedBlockingPool(size, factory, validator)
    }
```

### Async Blocking Queue

```scala
class ObjectPool[T]
    def take: Future[T]
    def giveBack(t: T): Future[ObjectPool[T]]
    
    def use(f: T => Future[B]): Future[B]
        val promise = Promise[B]()
        this.take(t => {
            try {
                f(t).map(result => 
                    giveBack(t).map(_ => promise.success(result))
            }
        }
        promise.future
```

支持使用

```scala
class Worker 
    val threadPool = new SingletonThreadPool()
    def doWork(f: => Unit)
        threadPool.execute(new Runnable {
            def run
                f
        }

class ThreadSafeObjectPool[T]
    Worker worker = new Worker
    Queue taskQueue = new Queue[Promise[T]]
    Queue[T] checkouts = new Queue[T]
    
    def take: Future[T] = {
        worker.work {
            val promsie = Promise[T]()
            if(checkouts.empty)
                promise.success(checkouts.dequeue)
            else
                taskQueue.enqueue(promise)
            promise.future
        }    
    }
    
    def giveBack(t: T): Future[ObjectPool[T]] = {
        worker.work {
            if(taskQueue.isEmpty)
                checkout.enqueue(t)
            else
                taskQueue.dequeue.success(t)
            Future(this)
        }
    }
```


### RateLimit 的实现

源码没看懂, 也没看到好的文章介绍, 我来分析下我自己的想法

首先, 当一个 request 到来以后, 假如 storedPermits 足够多的话, 直接执行 request。
如果没有足够的 permits, 但是 nextExecTime 小于等于当前时间的话, 说明只是 permits 不够
而已, 而不是因为 permits 被被人占用了。第三种情况是 nextExecTime 大于当前时间, 也就说
permits 已经被别人预约到未来的某个时间点了, 这个时候, 我们计算当前 request 需要继续
等多久才可以执行操作, wait(time) 等他醒来后可以直接执行操作, 和 permits 无关了, 下面是
伪代码

```acquire
void acquire(int permits)
    synchronized {
        // 计算可用的 permits
        long currentTime = system.currentNanoTime
        
        if(currentTime < nextExecTime) {
            // 当前和未来一段时间的 permits 都被预约了, 计算需要等待的时间
            // 当前的 storedPermits 肯定是 0 了
            int waitTime = permits / time4OnePermit
            nextExecTime = waitTime + nextExecTime // 更新下一次可执行的时间
            sleep(waitTime)
        } else { // 可用可用的 permits
            storedPermits += (currentTime - nextExecTime) / time4OnePermits
            nextExecTime = currentTime
            if(storedPermits > permits) { // 如果没有超出
                storedPermits -= permits
                // 继续执行后续的操作
            } else {
                int waitTime = (permits - storedPermits) / time4OnePermits
                nextExecTime = nextExecTime + waitTime
                storedPermits = 0 // 都用完了
                wait(waitTime)
            }
        }
    }
```

分布式 RateLimiter

只要把 nextExecTime 和 currentTime 放到 redis 中即可, 但是要注意锁的使用

**yahoo 开源项目**

[](https://yahooeng.tumblr.com/post/111288877956/cloud-bouncer-distributed-rate-limiting-at-yahoo)