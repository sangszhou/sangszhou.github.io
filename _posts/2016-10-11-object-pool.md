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


