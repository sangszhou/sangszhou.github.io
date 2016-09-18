---
layout: post
title:  "spray test analysis"
date:   "2016-08-29 17:50:00"
categories: scala
keywords: scala, concurrent
---

观察对象，一段 spray 代码：

```scala
Post("ur", HttpEntity(ContentTypes.`application/json`, jsonData)) .withHeaders(userHeader) ~> routes ~> check {
    assert(responseAs[String] == "hello")
	assert (status == ok)
}
```

这段代码看起来一目了然，逻辑表达的非常清晰，这说明spray test拥有优秀的 DSL。那么这么优秀的 DSL 是怎么实现的呢？

为了方便分析，把这段代码简化一下

```scala
Post ~> routes ~> check {
	assert(status == ok)
}
```

从结构上来看，这段代码分为四部分，分别是 Post, routes, check 和 block 内部的判断逻辑。但实际上，代码的关键部分只有两段，两个 ~> 方法。读这段代码需要 IDE 帮忙（intellij idea）, 借助 IDE 快速找到 scala implicit 函数定义。


###第一部分，Post 调用方法 ~>

Post() 实际上是调用 Post.apply()，转化成 new RequestBuilder(POST)，而 RequestBuilder 本身没有 ~> 方法，所以必然存在隐式转换，根据 IDE 的提示，找到隐式转换的定义

```scala
implicit class withTransformation2(HttpRequest) {
	def ~> (f: A => B) (implicit ta: TildeArrow): ta.Out = ta(request, f)
}
```
HttpRequest 隐式转换成了 withTransformation2 类型，然后调用它的 ~> 方法，这个方法的参数有两个，第一个是方法 A => B, 第二个是隐式参数 TildeArrow 类型。显然 routes 就是 f 参数，routes 的定义是

```scala
def routes: Route
type Route: RequestContext => Unit
```
可以推导出 A 是 RequestContext, B 是 Unit


第二个参数从哪里来呢？根据 IDE 对 TildeArrow 的定位，可以找到它的定义

```scala
abstract class TildeArrow[A, B] {
    type Out
    def apply(request: HttpRequest, f: A ⇒ B): Out
  }
```
然而，这个定义并没有说明 Out 是什么，再找它的子类实现

```scala
    implicit object InjectIntoRequestTransformer extends TildeArrow[HttpRequest, HttpRequest] {
      type Out = HttpRequest
      def apply(request: HttpRequest, f: HttpRequest ⇒ HttpRequest) = f(request)  
    }
```
它的唯一子类实现 A 和 B 都是 HttpRequest，和我们需要的不一样。借助 IDE 的支持也只能跟踪到这一步了，代码读不下去了。

相关的源代码又翻了一遍，找到这段

```scala
    implicit def injectIntoRoute(implicit timeout: RouteTestTimeout, settings: RoutingSettings,
                                 log: LoggingContext, eh: ExceptionHandler, defaultHostInfo: DefaultHostInfo) =

      new TildeArrow[RequestContext, Unit] {

        type Out = RouteResult

        def apply(request: HttpRequest, route: Route) = {
          val routeResult = new RouteResult(timeout.duration)
          val effectiveRequest =
            request.withEffectiveUri(
              securedConnection = defaultHostInfo.securedConnection,
              defaultHostHeader = defaultHostInfo.host)

          ExecutionDirectives.handleExceptions(eh orElse ExceptionHandler.default)(route) {
            RequestContext(
              request = effectiveRequest,
              responder = routeResult.handler,
              unmatchedPath = effectiveRequest.uri.path)
          }
          routeResult
        }
      }
  }
```

从 new 出来的实例泛型上看，它有些像我们需要的东西，然是这些参数又是从哪里来的呢？看不出从哪里来的，只能说是从 testcase 继承的类型中得到的吧。这里才知道 Out 是 RouteResult。其实从后面往前推得话，也知道需要的就是 RouteResult。

### 第二部分 RouteResult ~> check {xxx}

再根据 IDE 的找到第二个 ~> 的定义

```scala
def ~>[T](f: RouteResult ⇒ T): T = f(this)
```
那么问题来了，f 到底是 check 呢 还是 check {xxx}。其实 {} 里的东西是 check 的参数，对于参数的解析是要早于 ~> 方法的解析的，所以这里的 f 是 check {xxx}。

找到 check 的定义

```scala
def check[T](body: ⇒ T): RouteResult ⇒ T = result ⇒ dynRR.withValue(result.awaitResult)(body)

```
check 方法返回就是 RouteResult => T 即 ~> 的参数。check 本身的参数是 => T, 这也是一个方法，无参数方法， block 里的东西一般就是这种方法，实际上到现在我也没搞清楚无参数方法和一个值得区别是什么。


###想法

这样的话，这段代码就走通了。从上面的分析看出，读 scala 的代码是相当痛苦，读的过程不像 java 那样顺下来，就像这张图描述的一样

```
XXXXXX
	XXXXX
		XXXXXXX
			XXXXX
```		

读代码的过程要时刻提防可能出现的隐式转换，即使知道发生了隐式转换找到隐式转换的定义还得花一番功夫，就就像上面的例子一样为了找到隐式参数 TildeArrow[RequestContext, Unit] 花了好大功夫。除了隐式转换外，高阶函数，也就是函数作为参数带带来一定的复杂性，很多时候，为了代码的美观性，作者会把函数定义为一个值，就像 type Route = RequestContext => Unit，如果对 Route 不熟悉，还得去找一遍 Route 的定义，就算找到了定义还是越看越不习惯。

读框架的源码本来就有难度，在此之上有加上语言本身的难度，更难。此外，上面的代码分析还是比较粗糙的，有很多东西都没深究，比如产生隐式参数 TildeArraow 的代码是怎么引入到我们的测试用例的以及 TildeArrow 的写法是 cake pattern 都被忽略掉了。其实，上面的分析只是从类型上看明白了，代码的逻辑还没看呢。

Akka Actor 的代码也很难读，它的根本问题是 Actor 有 hireachy，呈现出一个树形关系，其实更多的时候是一个森林，如果对代码不熟悉很难把这个森林在脑海里构造出来，一个 Message 在 Actor 之间传递，传着传着就不知道传到哪里去了。另外一个问题是 Actor 没有类型信息，当我们拿到一个 ActorRef 时，我们只能通过名字猜测它的作用，它的定义，比如

```scala
def calculate (helper: ActorRef, info: SomeCaseClass) {
	helper.ask(info).mapTo[Int]
}
```
很难直接定位到 ActorRef 的定义，不能很快知道他对 SomeCaseClass 的处理逻辑。其实 actor 的这两个问题往往同时存在，当某个 ActorRef 作为另外一个 Actor 的成员变量时，要知道这个 ActorRef 的定义是比较麻烦的。

我觉得 Spray 和 Akka 的源代码都比较难读，我还是花了蛮多时间去尝试弄清楚它的逻辑，但是失败了。我也留意了下 Kafka 和 Spark 的源代码，发现这两个项目很少用隐式类型和类型系统，都在用 Java 的写法写 scala，不过采用了 scala 很多好用的算子，比如 map, filter 。至于这些算子的性能，就是另外一个问题了。回忆起去年我刚写 scala 的时候，追求的是纯函数式和完全异步，花了大量的时间选各种异步库，学习 async 语法，现在想来觉得并不是特别值得，还有可能给别人读代码，学习代码带来困难，老的代码又不能改，就出现光 ESClient 在一个项目里就有三套的问题，呃，就是知乎上那个实习生所鄙视的情况。选的一些库很多都不再更新，好像 spray 就不更新了，它的作者又搞了一套 web-service 框架，swagger 也不更新了，swagger 2.0 有蛮多吸引人的特性也只能远远看着，sbt的作者感觉也很高冷，issue 一堆也不回答。


我去年看过一篇文章，讲 scala 代码难读的问题，作者说为了弄明白一个 scalaz 的符号，花了大量的时间。当时我觉得这个作者功力不够，一个 scalaz 的符号不是分分钟就看出来了么。现在觉得，可能这只是作者举的一个例子吧，因为一个奇怪的符号的确会吓退很多不熟悉 scala 的开发者，让他们以为 scala 里有很多这种东西。上次，老大让我搞了一套 Oauth2 认证服务器，他表示应该尽量 stick to spray，我写了几张 PPT 给他讲我们为什么需要 Spring，用过之后感受到前所未有的安全感。
