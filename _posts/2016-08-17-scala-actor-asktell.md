---
layout: post
title:  "Actor ask tell"
date:   "2016-08-29 17:50:00"
categories: scala
keywords: scala, concurrent
---

### tell (!)

```scala
final def tell(msg: Any, sender: ActorRef): Unit = this.!(msg)(sender)
```

tell 函数是 actor 的核心，actor 是事件 (message) 驱动的，对 message 的处理逻辑都定义在 receive 函数中，只有当 message 到来时这些逻辑才会被调用。

```scala
// 用法
val actor = system.actorOf(Props[PingActor], "ping-actor")
actor ! "regards"
```
如果用 Intellij 查看 ! 函数的实现，IDE 会跳转到 ScalaActorRef

```scala
// If invoked from within an actor, then the actor reference is implicitly passed on as the implicit 'sender' argument
trait ScalaActorRef { ref: ActorRef =>
	def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit
}
```
在一个 scala trait 中看到 ref: ActorRef => 的写法，就意味着该 trait 的实现类必须也要继承 ActorRef，这算是一个依赖声明吧。

在 IDE 的帮助下，只能看到这一步了，找不到 ! 的具体实现。 intellij 对 scala 的支持还是不够 intelligent。其实 ! 函数最终会走到 Cell 的 sendMessage 方法上：

```scala
final def sendMessage(message: Any, sender: ActorRef): Unit =
    sendMessage(Envelope(message, sender, system))
```

然而，IDE 还是找不到 Cell.sendMessage 的具体实现

```scala
ActorCell extends Cell With dungeon.Dispatcher
```

sendMessage 的具体在 dungeon.Dispatcher

```scala
private[akka] trait Dispatch { this: ActorCell ⇒
  def sendMessage(msg: Envelope): Unit =
	try {
      dispatcher.dispatch(this, msg)
    } catch handleException
```

上面提到过 ref: ActorRef => 的用法，这里又碰到了，this: ActorCell => 道理一样，不过这边依赖的意图更加明显了。IDE 找不到 sendMessage 的实现也属正常，Scala 的继承关系太乱，是不好找。相比较而言 Java 就简单的多，Java 虽然支持 interface 的多重继承，但是这些 interface 仅提供函数声明，不提供函数实现，IDE 找函数的实现只会在一条继承链上。而 scala 对应的 trait 则不然，它能提供函数声明和实现，这样 IDE 找函数的实现就麻烦的多，一个类继承的 trait 中都可能有，当然也不是做不到，就看 IDE team 是不是愿意做了。

dispatcher 函数的定义在 MessageDispatcher 中，实现在子类 Dispatcher 中

```scala
protected[akka] def dispatch(receiver: ActorCell, invocation: Envelope):
    val mbox = receiver.mailbox
    mbox.enqueue(receiver.self, invocation)
    registerForExecution(mbox, true, false)
}

protected[akka] override def registerForExecution(mbox: Mailbox, hasMessageHint: Boolean, hasSystemMessageHint: Boolean): Boolean = {
    executorService execute mbox

private[akka] abstract class Mailbox(val messageQueue: MessageQueue)
  extends SystemMessageQueue with Runnable
```
maiBox 是 ActorCell 的成员变量（通过继承 Dispatcher 得来），它继承 Runnable 接口所以能放入 ExecutorService 中执行。我们说 actor ! Message 对消息的处理是异步的，其实直到 excutorService 之前，上面的所有逻辑还都发生在 sender 线程里，从这一行代码开始，sender 线程可以腾出手做下面的事了。这个 executorService 是配在 receiver 中的，和 sender 所在的线程池也没关系。

```scala
final def run = {
    processAllSystemMessages() //First, deal with any system messages
    processMailbox() //Then deal with messages

@tailrec private final def processMailbox(
    left: Int = java.lang.Math.max(dispatcher.throughput, 1),
    deadlineNs: Long = if (dispatcher.isThroughputDeadlineTimeDefined == true) System.nanoTime + dispatcher.throughputDeadlineTime.toNanos else 0L): Unit =
    if (shouldProcessMessage) {
      val next = dequeue()
      if (next ne null) {
        actor invoke next
        processAllSystemMessages()
        if ((left > 1) && ((dispatcher.isThroughputDeadlineTimeDefined == false) || (System.nanoTime - deadlineNs) < 0))
          processMailbox(left - 1, deadlineNs)
      }
    }    
```
MailBox 继承了 Runnable 接口，自然有 run 函数。run 是一个递归函数，每次调用处理一个消息，处理逻辑通过调用 invoke 函数实现

```scala
final def invoke(messageHandle: Envelope): Unit = try {
    currentMessage = messageHandle

    messageHandle.message match {
      case msg: AutoReceivedMessage ⇒ autoReceiveMessage(messageHandle)
      case msg                      ⇒ receiveMessage(msg)
    }
    currentMessage = null // reset current message after successful invocation
  } catch handleNonFatalOrInterruptedException { e ⇒
    handleInvokeFailure(Nil, e)
  } finally {
    checkReceiveTimeout // Reschedule receive timeout
  }

final def receiveMessage(msg: Any): Unit = actor.aroundReceive(behaviorStack.head, msg)  

```

对 PoisonKill, Terminate 系统消息的处理在 autoReceiveMessage 中，对普通消息的处理在 receiveMessage 中, behaviorStack 是一个 List[Actor.Receive]，因为 Actor 支持通过 become/unbecome 切换形态。而 Receive (PartialFunction[Any, Unit])函数就是我们写的对 message 的处理逻辑。


### ask (?)

actor 本身是没有 ask 函数的，如果想用 ask 函数，需要引入 akka.pattern.ask 依赖。Akka 官方并不推荐使用 ask 函数，因为它意味着处理 message 的 actor (receiver) 需要把处理结果返回 sender，这就引入了 sender 和 receiver 之间的依赖关系，本来 actor 之间都是各个独立存在的实体，因为 ask 函数引入了依赖会使程序变得复杂。但是在某些场景下 ask 函数会带来极大的便利性，所以它的存在还是有必要的。最终 akka 对 ask 的设计就像我们看到的一样，没有把 ask 作为 actor 的成员函数，表明自己对 ask 的不推荐态度，但又以隐式转换的方式支持它，表示如果你真的要用，我们仍提供这种 capability。

```scala
class PingActor extends Actor {
  override def receive: Receive = {
    case num: Int => sender() ! num+1
    case msg => println(msg)
    }
}

class testing extends FunSuite {

  val system = ActorSystem("testing")
  implicit val timeout = Timeout(3 second)
  import scala.concurrent.ExecutionContext.Implicits.global

  test("send arbitrary message to ping actor") {
    val pingActor = system.actorOf(Props[PingActor], "pingActor")

    pingActor ? "msg" onComplete {
      case Success(res) => println("ask success")
      case Failure(exp) => println("ask failure") // return failure
    }
    Thread.sleep(5000)
  }

  test("send num to ping actor") {
    val pingActor = system.actorOf(Props[PingActor], "pingActor")
    pingActor ? 1 onComplete {
      case Success(res) => println("ask success") // return success
      case Failure(exp) => println("ask failure")
    }
    Thread.sleep(5000)
  }
}
```

上面的例子展示了 ask (?) 函数的用法，两个相似的请求却有着不同的结果，写代码时要注意 ask message 时，这个 message 会返回结果。


那么 ask 函数是怎么实现的呢？简单来讲，就是创建了一个 proxy actor (PromiseActorRef)，它负责接受 receiver 处理后的返回值（假如 receiver 处理完消息后会返回给 sender 结果），proxy actor 拿到返回值后填充 Promise 变量，使 promise.future 可用。

```scala
// 只保留重要代码
actorRef ? Message

//implicit conversion
implicit def ask(actorRef: ActorRef): AskableActorRef = new AskableActorRef(actorRef)

AskableActorRef(val actorRef: ActorRef) extends AnyVal
	def ask(message: Any)(implicit timeout): Future[Any] =
		val a = PromiseActorRef(actorRef.provider, timeout)
		// 对 Promise 的调用在这里
		actorRef.tell(message, a)
		a.result.future
```
通过隐式转换把 ActorRef 类型转换成 AskableActorRef，其实它并不是 ActorRef 的子类，只是一个转换器而已。在 ask 函数创建 PromiseActorRef 并返回它的 promise.future，这是 promise 的典型用法，返回一个 “契约”。注意创建 PromiseActorRef 是通过 companion object。

```scala
PromiseActorRef
	def apply(provider: ActorRefProvider, timeout): PromiseActorRef
		result = Promise[Any]
		a = new PromiseActorRef(provider, result)
		implicit val ec = a.internalCallingThreadExecutionContext
		f = scheduler.scheduleOnce(timeout.duration)
			result tryComplete Failure(new AskTimeoutException()
		result.future onComplete { a.stop finally f.cancel)
		a

PromiseActorRef(provider: ActorRefProvider, result: Promise[Any])
	override def !(message)(implicit sender: ActorRef) =
		message match
			case Status.Success(r) => result.success(r)
			case Status.failure(r) => result.failure(r)
			case other => Success(other) // this one is what we got
```
ask 函数就是 ?, tell 函数就是 !. companion 在创建 PromiseActorRef 后启动定时器，超时后定时器会返回 Failure. PromiseActorRef 的 sender 函数很简单，就是填充 promise，填充完毕后 AskActorRef.ask 函数的返回值就有效了。注意，Status.Success 和 Status.failure 一般情况下应该用不着，大部分情况会走到 other 分支。

根据这个逻辑再回看最上面的 PingActor, 它在收到 num: Int 类型的数据后执行 sender() ! num+1，其实这个 sender() 就是 PromiseActorRef，这样逻辑就走通了。
