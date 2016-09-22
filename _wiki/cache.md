// 写的不错, 但是有些抽象, 看不懂
[volatile and memory barriers](http://jpbempel.blogspot.com/2013/05/volatile-and-memory-barriers.html)

一旦内存数据被推送到缓存, 就会有消息协议来确保所有的缓存会对所有的共享数据同步并保持一致, 这个是内存数据
对CPU核可见的技术成为内存屏障或者内存栅栏。

内存屏障提供两个功能。首先,它通过确保从另一个 CPU 来看屏障的两边, 所有的指令都是正确的执行顺序,
而保持顺序的外部可见性。其次他们可以实现内存可见性, 确保内存数据会同步到CPU缓存子系统。

**Store Barrier**

Store 屏障, 是 sfense 指令, 强制所有的 store 屏障指令之前的 store 指令, 都应在 store 屏障指令执行之前被执行,
并把 store 缓冲区的数据都刷到 cpu 缓冲。这使得程序对其他 cpu 可见, 这样其他 cpu 可以根据需要介入。一个好的例子是
Disruptor 中的 BatchEventProcessor, 当序列 sequence 被一个消费者更新时, 其他消费者和生产者知道该消费者的进度,
因此可以采用合适的动作, 所以屏障之前发生的内存更新都可见了。

store barrier 之前的 Store 操作都要在 barrier 之前完成并被打入主内存。

```java
private volatile long sequence = RingBuffer.INITAL_CURSOR_VALUE
T event = null
long nextSequence = sequence.get() + 1L

while(running)
    try
        final long availableSequence = barrier.waitFor(nextSequence)
        while(nextSequence <= availableSequence)
            event = ringBuffer.get(nextSequence)
            boolean endOfBatch = nextSequence == availableSequence
            eventHandler.onEvent(event, nextSequence, endOfBatch)
            nextSequence ++
            // store barrier inserted here!!!
        sequence.set(nextSequence - 1L)
    catch
        exceptionHandler.handle(ex, nextSequence, event)
        sequence.set(nextSequence)
        // store barrier inserted here!!!
        nextSequence ++
```
**Load Barrier**

Load 屏障, ifense 指令, 强制所有的 Load 屏障指令之后的 Load 指令, 都在该 load 屏障指令执行之后被执行,
并且一直等到 load 缓冲区被该 cpu 读完之后的才能执行 load 命令。这使得其他 cpu 暴露出来的程序状态对该 cpu 可见,
这之后的 cpu 可以进行后续处理。

load barrier 之后的 load 操作都要在 barrier 之后执行, 不得重排序到 barrier 之前。

如果 store barrier 和 load barrier 之间有先后关系, 那么数据就保证可见了

如果只有 store barrier 没有 load barrier, 就像 final 且假设 final 可以修改, 那么可能出现这种情况

```
1: a = 2 // 假设 final a 可以被修改
1: store barrier
2:                  read a
```
这个时候, 因为没有 load barrier 的读缓冲保护, read a 可能读到的是缓冲区的脏数据

那为什么 final 在构造函数完毕之后就没问题呢? 这是因为第一次读 final 变量时, 缓冲区还没有, 有了以后就一直是对的了, 不会 stale
。其实理解的还是不到位, final 字段在构造函数完成之前不会暴露出来, 如果暴露出来了, final 肯定完成创建, 其他线程看到的肯定是实际值。
和普通的字段有什么区别呢?

**Full Barrier**

mfense, 复合了 load 和 save 屏障功能

java 内存模型中的 volatile 变量在写操作之后插入 store 屏障, 在读操作以前插入 load 指令, 一个类的 final 字段在初始化之后
插入 store 屏障, 来确保 final 字段在构造函数初始化完成并可被使用时可见

除此之外还有其他内存屏障, 数据依赖屏障和 lock/unlock 屏障

> mfence：串行化发生在mfence指令之前的读写操作

> lfence：串行化发生在mfence指令之前的读操作、但不影响写操作

> sfence：串行化发生在mfence指令之前的写操作、但不影响读操作


## String

equals functions

1. 比较指针是否一致, 如果一致直接返回 true
2. 比较参数类型是否是 string, 不是直接返回 false
3. 比较 char[] 的 length, 不相等返回 false
4. 逐个 char 比较, 不想同返回 false

hashCode

第一次调用时确定其值, 内容修改其值不变

```java
    public int hashCode() {
        int var1 = this.hash;
        if(var1 == 0 && this.value.length > 0) {
            char[] var2 = this.value;

            for(int var3 = 0; var3 < this.value.length; ++var3) {
                var1 = 31 * var1 + var2[var3];
            }
            this.hash = var1;
        }
        return var1;
    }
```

