## 生产者消费者问题

生产者消费者问题是研究多线程程序时绕不开的经典问题之一，它描述是有一块缓冲区作为仓库，生产者可以将产品放入仓库，
消费者则可以从仓库中取走产品。解决生产者/消费者问题的方法可分为两类：

1. 采用某种机制保护生产者和消费者之间的同步；

2. 在生产者和消费者之间建立一个管道。第一种方式有较高的效率，并且易于实现，代码的可控制性较好，属于常用的模式。

第二种管道缓冲区不易控制，被传输数据对象不易于封装等，实用性不强。因此本文只介绍同步机制实现的生产者/消费者问题。


同步问题核心在于：如何保证同一资源被多个线程并发访问时的完整性。常用的同步方法是采用信号或加锁机制，
保证资源在任意时刻至多被一个线程访问。Java语言在多线程编程上实现了完全对象化，提供了对同步机制的良好支持。
在Java中一共有四种方法支持同步，其中前三个是同步方法，一个是管道方法。

1. wait() / notify()方法
2. await() / signal()方法
3. BlockingQueue阻塞队列方法
4. PipedInputStream / PipedOutputStream


### await/ signal 方法

在JDK5.0之后，Java提供了更加健壮的线程处理机制，包括同步、锁定、线程池等，它们可以实现更细粒度的线程控制。
await()和signal()就是其中用来做同步的两种方法，它们的功能基本上和wait() / nofity()相同，完全可以取代它们，
但是它们和新引入的锁定机制Lock直接挂钩，具有更大的灵活性。通过在Lock对象上调用newCondition()方法，将条件变量和一个锁对象进行绑定，
进而控制并发程序访问竞争资源的安全

我觉得下面这个写法是有缺陷的, 考虑当 MAX_SIZE 等于 5, list.size = 1, 此时 Producer 想要一下生产 5 个, consumer 想要一下使用 2 个,
这个时候, 程序就锁住了, 更好的办法是有一个可用的就取一个, 有一个空位就放一个

```java
class Storage
    List list = new LinkedList
    
    Lock lock = new ReentrantLock
    Condition full = lock.newCondition
    Condition empty = lock.newCondition
    
    public void produce(int num)
        lock.lock
        while(list.size + num > MAX_SIZE)
            full.await
        
        for(int i = 0; i < num; i++)
            list.add(new Object)
            
        empty.signalAll
        
        lock.unlock
    
    public void consumer(int num)
        lock.lock
        while(list.size < num)
            empty.await
        
        for(int i = 0; i < num; i++)
            list.remove
         
        full.signalAll
        // 不需要 empty.signalAll
        lock.unlock

```

### wait() / notify()方法

wait() / nofity()方法是基类Object的两个方法，也就意味着所有Java类都会拥有这两个方法，这样，我们就可以为任何对象实现同步机制。

wait()方法：当缓冲区已满/空时，生产者/消费者线程停止自己的执行，放弃锁，使自己处于等等状态，让其他线程执行。

notify()方法：当生产者/消费者向缓冲区放入/取出一个产品时，向其他等待的线程发出可执行的通知，同时放弃锁，使自己处于等待状态。

```java
class Producer implements Runnable
    public Producer(List list) { this.list = list }
    List list;
    
    public void produce(int num)
        synchronized(list) {
            while(list.size + num > MAX_SIZE)
                try list.wait
                catch (InterruptedException)
            
            for(int i = 0; i < num; i ++)
                list.add(new Object)
            
            list.notifyAll // 缺点就是有些 consumer 线程起来发现资源已经没了, 然后再把自己 lock 住
        }

class Consumer implements Runnable 
    public Consumer(List list) { this.list = this }
    List list;
    
    public void consume(int num)
        synchronized(list)
            while(list.size < num)
                list.wait
            
            for(int i = 0; i < num; i ++)
                list.remove
            
            list.notifyAll
```

### Blocking Queue

阻塞队列实现生产者消费者模式超级简单，它提供开箱即用支持阻塞的方法put()和take()，
开发者不需要写困惑的wait-nofity代码去实现通信。BlockingQueue 一个接口，
Java5提供了不同的现实，如ArrayBlockingQueue和LinkedBlockingQueue，两者都是先进先出（FIFO）顺序。
而ArrayLinkedQueue是自然有界的，LinkedBlockingQueue可选的边界。下面这是一个完整的生产者消费者代码例子，
对比传统的wait、nofity代码，它更易于理解。

```java
class Producer implements Runnable {
 
    private final BlockingQueue sharedQueue;
 
    public Producer(BlockingQueue sharedQueue) {
        this.sharedQueue = sharedQueue;
    }
 
    @Override
    public void run() {
        for(int i=0; i<10; i++){
            try {
                System.out.println("Produced: " + i);
                sharedQueue.put(i);
            } catch (InterruptedException ex) {
                Logger.getLogger(Producer.class.getName()).log(Level.SEVERE, null, ex);
            }
        }
    }
}

class Consumer implements Runnable{
 
    private final BlockingQueue sharedQueue;
 
    public Consumer (BlockingQueue sharedQueue) {
        this.sharedQueue = sharedQueue;
    }
 
    @Override
    public void run() {
        while(true){
            try {
                System.out.println("Consumed: "+ sharedQueue.take());
            } catch (InterruptedException ex) {
                Logger.getLogger(Consumer.class.getName()).log(Level.SEVERE, null, ex);
            }
        }
    }
}

public class ProducerConsumerPattern {
 
    public static void main(String args[]){
 
     //Creating shared object
     BlockingQueue sharedQueue = new LinkedBlockingQueue();
 
     //Creating Producer and Consumer Thread
     Thread prodThread = new Thread(new Producer(sharedQueue));
     Thread consThread = new Thread(new Consumer(sharedQueue));
 
     //Starting producer and Consumer thread
     prodThread.start();
     consThread.start();
    }
}
```


## CAS 总结

三段代码, 分别是 CAS 计数器, CAS 堆栈和 CAS 链表

### CAS 计数器

```java
    public class CasCounter implements Serializable {
        private static Unsafe unsafe;
        private static long valueOffset;

        static {
            try {
                Field field = Unsafe.class.getDeclaredField("theUnsafe");
                field.setAccessible(true);
                unsafe = (Unsafe) field.get(Unsafe.class);
            } catch (Exception e) {
                e.printStackTrace();
            }
            try {
                valueOffset = unsafe.objectFieldOffset(CasCounter.class.getDeclaredField("value"));
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            }
        }

        private volatile int value;

        public CasCounter() {
            value = 0;
        }

        public CasCounter(int initValue) {
            this.value = initValue;
        }

        public int getValue() {
            return value;
        }

        //线程安全
        public int increment(int incrNum) {         
            while (true) {
                int oleVal = value;
                int newVal = oleVal + incrNum;
                if (unsafe.compareAndSwapInt(this, valueOffset, oleVal, newVal)) {
                    return newVal;
                }
            }
        }

        //非线程安全
        public int incrementNo(int incrNum) {        
            value += incrNum;
            return value;
        }
    }
```

可以使用 AtomicInteger 模拟 CAS 操作, 之所以说是模拟, 因为它已经有了 increment 方法

```java
public class NonblockingCounter {
    private AtomicInteger value;
    public int getValue() {
        return value.get();
    }
    public int increment() {
        int v;
        do {
            v = value.get();
        while (!value.compareAndSet(v, v + 1));
        return v + 1;
    }
}
```


### CAS 堆栈

```java
public class ConcurrentStack<E> {
    AtomicReference<Node<E>> head = new AtomicReference<Node<E>>();
    
    public void push(E item) {
        Node<E> newHead = new Node<E>(item);
        Node<E> oldHead;
        do {
            oldHead = head.get();
            newHead.next = oldHead;
        } while (!head.compareAndSet(oldHead, newHead));
    }

    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;
        do {
            oldHead = head.get();
            if (oldHead == null) 
                return null;
            newHead = oldHead.next;
        } while (!head.compareAndSet(oldHead,newHead));
        return oldHead.item;
    }
    
    static class Node<E> {
        final E item;
        Node<E> next;
        public Node(E item) { this.item = item; }
    }
}
```

**性能考虑**

在轻度到中度的争用情况下，非阻塞算法的性能会超越阻塞算法，因为 CAS 的多数时间都在第一次尝试时就成功，
而发生争用时的开销也不涉及线程挂起和上下文切换，只多了几个循环迭代。没有争用的 CAS 要比没有争用的锁便宜得多
（这句话肯定是真的，因为没有争用的锁涉及 CAS 加上额外的处理），而争用的 CAS 比争用的锁获取涉及更短的延迟。

在高度争用的情况下（即有多个线程不断争用一个内存位置的时候），基于锁的算法开始提供比非阻塞算法更好的吞吐率，
因为当线程阻塞时，它就会停止争用，耐心地等候轮到自己，从而避免了进一步争用。但是，这么高的争用程度并不常见，
因为多数时候，线程会把线程本地的计算与争用共享数据的操作分开，从而给其他线程使用共享数据的机会。（
这么高的争用程度也表明需要重新检查算法，朝着更少共享数据的方向努力。）“流行的原子” 中的图在这方面就有点儿让人困惑，
因为被测量的程序中发生的争用极其密集，看起来即使对数量很少的线程，锁定也是更好的解决方案。

### 非阻塞链表

目前为止的示例（计数器和堆栈）都是非常简单的非阻塞算法，一旦掌握了在循环中使用 CAS，就可以容易地模仿它们。
对于更复杂的数据结构，非阻塞算法要比这些简单示例复杂得多，因为修改链表、树或哈希表可能涉及对多个指针的更新。
CAS 支持对单一指针的原子性条件更新，但是不支持两个以上的指针。所以，要构建一个非阻塞的链表、树或哈希表，需要找到一种方式，
可以用 CAS 更新多个指针，同时不会让数据结构处于不一致的状态。

在链表的尾部插入元素，通常涉及对两个指针的更新：“尾” 指针总是指向列表中的最后一个元素，
“下一个” 指针从过去的最后一个元素指向新插入的元素。因为需要更新两个指针，所以需要两个 CAS。
在独立的 CAS 中更新两个指针带来了两个需要考虑的潜在问题：**如果第一个 CAS 成功，而第二个 CAS 失败，会发生什么？**
如果其他线程在第一个和第二个 CAS 之间企图访问链表，会发生什么？

**帮助**

对于非复杂数据结构，构建非阻塞算法的 “技巧” 是确保数据结构总处于一致的状态（甚至包括在线程开始修改数据结构和它完成修改之间），
还要确保其他线程不仅能够判断出第一个线程已经完成了更新还是处在更新的中途，还能够判断出如果第一个线程走向 AWOL，完成更新还需要
什么操作。如果线程发现了处在更新中途的数据结构，它就可以 “帮助” 正在执行更新的线程完成更新，然后再进行自己的操作。当第一
个线程回来试图完成自己的更新时，会发现不再需要了，返回即可，因为 CAS 会检测到帮助线程的干预（在这种情况下，是建设性的干预）。

这种 “帮助邻居” 的要求，对于让数据结构免受单个线程失败的影响，是必需的。如果线程发现数据结构正
处在被其他线程更新的中途，然后就等候其他线程完成更新，那么如果其他线程在操作中途失败，这个线程就可能永
远等候下去。即使不出现故障，这种方式也会提供糟糕的性能，因为新到达的线程必须放弃处理器，导致上下文切换，或者等到自己的时间片过期（而这更糟）。

```java
public class LinkedQueue <E> {
    private static class Node <E> {
        final E item;
        final AtomicReference<Node<E>> next;
        Node(E item, Node<E> next) {
            this.item = item;
            this.next = new AtomicReference<Node<E>>(next);
        }
    }
    private AtomicReference<Node<E>> head
        = new AtomicReference<Node<E>>(new Node<E>(null, null));
    private AtomicReference<Node<E>> tail = head;
    public boolean put(E item) {
        Node<E> newNode = new Node<E>(item, null);
        while (true) {
            Node<E> curTail = tail.get();
            Node<E> residue = curTail.next.get();
            if (curTail == tail.get()) { // 第一个 CAS
                if (residue == null) /* A */ {
                    if (curTail.next.compareAndSet(null, newNode)) /* C */ {
                        tail.compareAndSet(curTail, newNode) /* D */ ;
                        return true;
                    }
                } else {
                    tail.compareAndSet(curTail, residue) /* B */;
                }
            }
        }
    }
}
```

涉及到两个指针, 分别是 tail 和 tail.next. 线程 T1 先更新 tail.next, 进入 C 下面的 block.

考虑此时另一个线程 T2 进来了, residue 不为 null, 它会执行 B, 也就是和 D 一样的代码, 所以 D 不需要再判断
是不是成功执行了, 直接返回 true
