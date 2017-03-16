---
layout: post
title:  "Guava rate limit"
date:   "2016-03-10 17:50:00"
categories: architecture
keywords: architecture
---

## Guava rate limit Impl

guava 提供了两种 rate limit 的实现类，分别是 SmoothBursty 和 SmoothWarmingUp. SmoothBursty 比较简单，它的实现思路就是当消息到来时根据自己已有的 permits 和 permits 的
生成速度判断请求线程是否需要等待 permits 以及等待多久。而 SmoothWarmingUp 则比较复杂，看了一个上午也没太理解它的算法目的和算法计算方式，在代码中有很多的注释解释这个算法，但是就是
看不懂，这里就只拿第一种 SmoothBursty 举例子

```java
  static final class SmoothBursty extends SmoothRateLimiter {
    /** The work (permits) of how many seconds can be saved up if this RateLimiter is unused? */
    final double maxBurstSeconds;

    SmoothBursty(SleepingStopwatch stopwatch, double maxBurstSeconds) {
      super(stopwatch);
      this.maxBurstSeconds = maxBurstSeconds;
    }

    @Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
      double oldMaxPermits = this.maxPermits;
      maxPermits = maxBurstSeconds * permitsPerSecond;
      if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = maxPermits;
      } else {
        storedPermits =
            (oldMaxPermits == 0.0)
                ? 0.0 // initial state
                : storedPermits * maxPermits / oldMaxPermits;
      }
    }

    @Override
    long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
      return 0L;
    }

    @Override
    double coolDownIntervalMicros() {
      return stableIntervalMicros;
    }
  }
```

SmoothBursty 的 storedPermitsToWaitTime 函数直接返回 0，也就是当 permits 可用时不需要等，而 SmoothWarmingUp 在 permit 可用的情况下仍然需要等待，这种策略的背景是有些工作需要提前做。
一般的场景好像也用不着找等待。

再说 doSetRate 函数，这个函数的第二个参数并没有用到。这个函数用来计算 maxPermits 以及更新 storedPermits

再看 RateLimit 自己的接口

```java
public abstract class RateLimiter {
    // createWithCapacity(permitsPerSecond, 1, TimeUnit.SECONDS)}".
    public static RateLimiter create(double permitsPerSecond) {}
    public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit) {
        return create(SleepingStopwatch.createFromSystemTimer(), permitsPerSecond, warmupPeriod, unit, 3.0);
    }
    
    public final void setRate(double permitsPerSecond) {
        synchronized (mutex()) {
            doSetRate(permitsPerSecond, stopwatch.readMicros());
        }
    }
    
    public double acquire() {
        return acquire(1);
    }
    public double acquire(int permits) {
        long microsToWait = reserve(permits);
        stopwatch.sleepMicrosUninterruptibly(microsToWait);
        return 1.0 * microsToWait / SECONDS.toMicros(1L);
    }
    
    public boolean tryAcquire(long timeout, TimeUnit unit) {
        return tryAcquire(1, timeout, unit);
    }
    
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
        long timeoutMicros = max(unit.toMicros(timeout), 0);
        checkPermits(permits);
        long microsToWait;
        synchronized (mutex()) {
          long nowMicros = stopwatch.readMicros();
          if (!canAcquire(nowMicros, timeoutMicros)) {
            return false;
          } else {
            microsToWait = reserveAndGetWaitLength(permits, nowMicros);
          }
        }
        stopwatch.sleepMicrosUninterruptibly(microsToWait);
        return true;
    }
}
```

SleepingStopwatch 非常简单，等待就是调用 Thread.sleep 函数。因为 permits 的到来是可以计算出来的，没有谁会提前叫醒这个线程。

在 SmoothRateLimit 中有最重要的两个函数，这两个函数也比较简单

```java
  final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);

    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }
  
  void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      storedPermits = min(maxPermits, storedPermits + newPermits);
      nextFreeTicketMicros = nowMicros;
    }
  }
```

resync 只有在请求到来才会执行，执行的结果是计算出 storedPermits 的数量，因为 Bursty 是有最大值的，所以不能超过最大值，然后把 nextFreeTicketMicros 更新为当前时间。

当一个请求到来时，如果没有足够的 permits, 请求会查看 nextFreeTicketMicros 和当前时间的关系，如果小于等于当前时间，则这个请求会预支未来的 permits，如果大于的话，则会
阻塞当前请求，并 reserve 一些 permits, 然后自己进入 sleep 状态。

### 问题

RateLimit 的创建时通过工厂方法来的，SmoothBursty RateLimit 最多只能保存 1 秒的 permits, SmoothBursty 又是私有类，没有办法在包外访问，这也是比较烦的点。我看到已经有 issue 
提出开放这个接口，但是至今 21-rc2 没有看到开放出来的接口。