---
layout: post
title: scala concurrent
categories: [scala]
keywords: scala
---

## Scala class hierarchy

![](/images/posts/scala/classhierarchy.png)



## Actor

### 生命周期


### 特殊的 Actor

PromiseActorRef, DeadLetterActorRef, EmptyLocalActorRef

## Future 的挂载点配置

### 配置文件声明访问 ES 的连接池

```
es-dispatcher {
    type = dispatcher
    executor = "thread-pool-executor"
    thread-pool-executor {
        core-pool-size-min = 2
        core-pool-size-factor = 4.0
        core-pool-size-max = 16
    }
}
```

其中 core-pool-size-factor 是按照 CPU 个数来确定的, 线程池的最大允许工作线程数是

`max(size-max, #core * size-factor)`

Threadpool 有两种, fork-join-pool 和 thread-pool-executor

Dispatcher 默认有 4 种

> Dispatcher: 默认的, 事件驱动, 将一组 actor 绑定到一个线程组中, 共享线程

> PinnedDispatcher: 一个 actor 一个线程, 线程不共享

> BalancingDispatcher: 所有的 actor 共享一个 mailbox, 任务繁重的 actor 会把一些任务分派到空闲的 actor 上

> CallingThreadDispatcher: 仅在当前线程执行 invocation, 用于测试

default-mailbox: 无限大的邮箱 

### 获取配置文件中的线程池

```scala
def findByName(name: String) = Boot.system.dispatcher.lookup(name)

val dispacher = findByname("es-dispatcher")

implicit val executionContext = DispatcherUtil.esDispatcher
```

这样在 implicit 变量下面的 Future 都会默认使用 esDispatcher, 如果觉得粒度太粗的话, 可以直接把
implicit 变量放到 Future 的参数列表中

```  
def apply[T](body: =>T)(implicit executor: ExecutionContext): Future[T] = impl.Future(body)

val f = Future {
  Thread.sleep(5000)
  "future response"
}(Dispatchers.netdispatcher)
```

## Actor 的挂载点

Actor 有两种挂载点，第一种是以 route 的方式存在，挂载在一个 router (dispatcher) 上，另一种是挂载在 actor 上

### 配置 actor 线程池

```
akka.actor.deployment {
    "/service/dlp-fetch" { // 他用的是哪个 dispatcher
        router = round-robin-pool
        nr-of-instance = 5
    }
    
    "/service/bug-fetch-actor/*" {
        dispatcher = user-info-dispatcher
    }
    
    "/policy-actor" {
      router = smallest-mailbox
      nr-of-instances = 8
    }
}
```

### 使用 配置的 Actor 线程池 (还需要补充)

对于 router 的挂载方式，在创建 Actor 时，需要 .fromRoute 手动声明，但是 dispatcher 这种，只要 actor 的名字符合即可

```
val PolicyActor = localSystem.actorOf(Props[PolicyActor].withRouter(FromConfig), "policy-actor")
```

## Akka default dispatcher


```
default-dispatcher {
    type = "Dispatcher"
    executor = "fork-join-executor"
    
    fork-join-executor {
        parallelism-min: 8
        parallelism-factor: 3.0
        parallelism-max: 64
    }
}
```

