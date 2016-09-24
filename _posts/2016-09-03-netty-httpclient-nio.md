---
layout: post
title:  "Netty Http Client and NIO"
date:   "2016-09-03 00:00:00"
categories: [nio]
keywords: java, netty, nio
---


### 背景

希望通过 9200 端口访问 Elasticsearch server, 这样当 ES 返回的结果和期望的不一致时可以根据 ES 的 log debug,
但是这样就需要自己写 Http client, 官方的 client 就没法用了

在编写 HttpClient 时, 参考了 async mysql client, 因为他们都具有一个特点, 那就是每个 Connection 上同时只能有一次数据发送的过程

### ConnectionPool

```scala
trait AsyncObjectPool[T] {
  def take: Future[T]
  def giveBack(item: T): Future[AsyncObjectPool[T]]
  def close: Future[AsyncObjectPool[T]]

  def use[A](f: T => Future[A])(implicit executionContext: ExecutionContext): Future[A] = {
    take.flatMap { item => {
      val p = Promise[A]()
      // calling f might throw exception
      try {
        f(item).onComplete { res =>
          giveBack(item).onComplete { _ => p.complete(res) }
      } catch {
        case error: Throwable =>
          giveBack(item).onComplete { _ => p.failure(error) }
      p.future
}
```

use 函数可以看成是 loan pattern, 输入是一个函数, f: T => Future[A], 在函数内部使用对象池的资源,
最后再回收资源

AsyncObjectPool 不是线程安全的, 这是它线程安全的子类

```scala
class ThreadSafeObjectPool[T](factory: ObjectFactory[T]) extends AsyncObjectPool[T] {
    private val mainPool = Worker()
    private val waitQueue = new Queue[Promise[T]]()
    private val checkouts = new ArrayBuffer[T](10)
    
    // available connections
    private val poolable = new Stack[T]()

    // get object from object pool
    private def checkout(promise: Promise[T]) = {
        this.mainPool.action {
          if (this.isFull) this.enqueuePromise(promise)
          else this.createOrReturnItem(promise)
    
    override def take: Future[T] = {
        if(this.closed) Promise.failed(new Exception("failed to take item from pool")).future
        else
          val promise = Promise[T]()
          this.checkout(promise)
          promise.future
    
    private def addBack(item: T, promise: Promise[AsyncObjectPool[T]]) = {
        this.poolable.push(item)
        if (this.waitQueue.nonEmpty) this.checkout(this.waitQueue.dequeue())
        promise.success(this)

    def giveBack(item: T): Future[AsyncObjectPool[T]] = {

        val promise = Promise[AsyncObjectPool[T]]

        this.mainPool.action
          val idx = this.checkouts.indexOf(item)
          this.checkouts.remove(idx)
          this.factory.validate(item) match {
            case Success(item) => this.addBack(item, promise)
            case Failure(e) => {
              this.factory.destroy(item)
              promise.failure(e)
        
        promise.future
```

为了线程安全, 可以使用 synchronized, 单线程或者转化为生产者消费者问题。synchronized 是一个好的选择,
写起来简单, 但是每次调用函数都要上锁, 代价稍高, 且不能写成异步的。单线程是一个更优秀的解决方法, 可以写成
异步的, 还不用锁, 但是难度大大一些, 需要仔细考虑哪些函数内部需要用到 mainPool.action 那些可以不用, 每次
调用带有 mainPool.action 的函数都伴随着一次线程的上下文切换, 太多也不好。

接下来就是把对象池实例化, 转化为连接池

```scala
class ConnectionPoolThreadSafe[T <: Connection](
                                       factory: ObjectFactory[T],
                                       executionContext: ExecutionContext = ExecutorServiceUtils.CachedExecutionContext
                                     )
  extends ThreadSafeObjectPool[T](factory) with Connection {
    
  def sendQuery(query: HttpRequest): Future[HttpResponse] = this.use(_.sendQuery(query))(executionContext)
```

连接池的用法有两种, 一种是通过 get 函数返回它持有的连接, 获得连接的 client 需要在用完以后释放, 另一种是连接池直接继承连接, 它本身也提供
网络访问服务, ConnectionPoolThreadSafe 使用的是第二种

### Connection

Connection 是用于发送消息获得结果的工具, 因此重要的函数只有一个, sendQuery

```scala
trait Connection
  def disconnect: Future[Connection]
  def connect: Future[Connection]
  def isConnected: Boolean
  def sendQuery(query: HttpRequest): Future[HttpResponse]

trait EventConnectionDelegate
  def connected(ctx: ChannelHandlerContext)
  def onMessageReceived(result: HttpResponse)
  def exceptionCaught(exception: Throwable)
```

为了让 client 发送消息后能拿到对应的 response, 需要 http connection 记得自己发出去的 request 返回的时候结果应该给谁,
也就是说, 每次 http connection 只能处理一个请求, 当请求返回时, 直接送给挂在自己身上的 client, 当然这是通过 Promise 传递的

```scala
class HttpConnection(configuration: Configuration,
                      group: EventLoopGroup = NettyUtils.DefaultEventLoopGroup,
                      implicit val executionContext: ExecutionContext = ExecutorServiceUtils.CachedExecutionContext)

  extends Connection with EventConnectionDelegate {

      // 结果到来后, 需要填充的对象, client 持有它的索引
      private val queryPromiseReference = new AtomicReference[Option[Promise[HttpResponse]]](None)
    
      // http connection, 用于发送消息的载体, this 变量传递进去, 是为了当结果返回时, 调用自己的 onMessageReceived 方法
      private final val connectionHandler = new HttpConnectionHandler(
        this,
        configuration,
        group,
        executionContext,
        connectionId
      )
      
      // 返回值是给 client 的引用
      def sendQuery(query: HttpRequest): Future[HttpResponse] = {
        this.validateIsReadyForQuery
        val promise = Promise[HttpResponse] // 新创建一个 promise
        this.setQueryPromise(promise) // 填充 promise, 等待结果
        this.connectionHandler.write(query)
        promise.future
      }
      
      override def connect: Future[Connection] = {
        this.connectionHandler.connect.onFailure {
          case e => this.connectionPromise.tryFailure(e)
        }
        this.connectionPromise.future
      }
    
      override def onMessageReceived(result: HttpResponse): Unit = {
        succeedQueryPromise(result)
      }
    
      private def succeedQueryPromise(queryResult: HttpResponse) = 
        this.clearQueryPromise.foreach 
          _.success(queryResult) // 消息到来, 填充 promise
```

### Netty 发送消息

```scala

class HttpConnectionHandler(
                            eventConnectionDelegate: EventConnectionDelegate,
                            configuration: Configuration,
                            group: EventLoopGroup,
                            executionContext: ExecutionContext,
                            connectionId: String) extends SimpleChannelInboundHandler[HttpResponse] {

  private implicit val internalPool = executionContext
  private final val log = LoggerFactory.getLogger(s"[connection-handler]${connectionId}")
  private final val bootstrap = new Bootstrap().group(this.group) // Bootstrap group 每个 server 只有一个?
  private final val connectionPromise = Promise[HttpConnectionHandler]
    
  def connect: Future[HttpConnectionHandler] = {
      this.bootstrap.channel(classOf[NioSocketChannel])
      this.bootstrap.handler(new ChannelInitializer[io.netty.channel.Channel]() {
        override def initChannel(ch: Channel): Unit = {
          ch.pipeline().addLast(
            new HttpClientCodec(),
            new HttpContentDecompressor(),
            new HttpObjectAggregator(512 * 1024),
            HttpConnectionHandler.this) // 把自己加到 handler 列表中
        }
      })
   
   // 创建 节点
   this.bootstrap.connect(new InetSocketAddress(configuration.host, configuration.port)).onFailure {
      case exception => this.connectionPromise.tryFailure(exception)
    }
    
   // 接受逻辑
   
   override def channelRead0(ctx: ChannelHandlerContext, msg: HttpResponse): Unit = {
       this.eventConnectionDelegate.onMessageReceived(msg)
   }

  
   // 发送逻辑 
   def write(req: HttpRequest): ChannelFuture = {
       writeAndHandleError(req)
     }
   
   private def writeAndHandleError(message: Any): ChannelFuture = {
       if(this.currentContext.channel().isActive) {
         val res = this.currentContext.channel().writeAndFlush(message)
         res.onFailure {
           case e: Throwable => handleException(e)
         }
         res
       } else {
         val error = new Exception("This channel is not active any more")
         handleException(error)
         this.currentContext.channel().newFailedFuture(error)
       }
     } 
```

### 其他, 

**回调函数转 Future**

```scala
object ChannelFutureTransformer {

  implicit def toFuture(channelFuture: ChannelFuture): Future[ChannelFuture] = {
    val promise = Promise[ChannelFuture]

    channelFuture.addListener(new ChannelFutureListener {
      override def operationComplete(future: ChannelFuture): Unit =
        future.isSuccess match 
          case true => promise.success(future)
          case false =>
            val exception = if (future.cause() == null) {
              new Exception("canceled channel future exception")
            } else future.cause()
            promise.failure(exception)
    promise
```

### 对象工厂

```scala

trait ObjectFactory[T]
  def create: T
  def destroy(item: T)
  def validate(item: T): Try[T]
  def test(item: T): Try[T] = validate(item)

class HttpConnectionFactory(configuration: Configuration) extends ObjectFactory[HttpConnection] {

  override def create: HttpConnection = {
    val connection = new HttpConnection(configuration)
    FutureUtils.awaitFuture(connection.connect)
    connection

  def destroy(item: HttpConnection): Unit =
    try {   item.disconnect
  } catch {
    case e: Exception =>
      log.error("Failed to close the connection", e)
```

### More

应该能提供配置, 设置最大最小连接数之类的逻辑, 目前这个连接池, 每个只能连一个节点

```scala
trait ObjectPoolControl[T]
  def setMaxTotal(max: Int)
  def getMaxTotal: Int
  def setDefaultMaxPerRoute(max: Int)
  def getDefaultMaxPerRoute: Int
  def setMaxPerRoute(route: T, max: Int)
  def getMaxPerRoute(route: T): Int
```

这部分代码的设计, 可以参考 Spring RestTemplate 的实现