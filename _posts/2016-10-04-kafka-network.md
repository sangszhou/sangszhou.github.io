## Socket Server 

![](/images/posts/kafka/kafka_broker_internals.png)

SocketServer是一个 NIO 的服务器, 它的线程模型:

1. 一个Acceptor线程接受/处理所有的新连接
2. N个Processor线程,每个Processor都有自己的selector,从每个连接中读取请求
3. M个Handler线程处理请求,并将产生的请求返回给Processor线程用于写回客户端


SocketServer在启动时(Kafka->KafkaServer),会启动一个Acceptor和N个Processor.

```scala
def startup() {
    val brokerId = config.brokerId
    var processorBeginIndex = 0
    
    endpoints.values.foreach { endpoint =>
      val protocol = endpoint.protocolType
      val processorEndIndex = processorBeginIndex + numProcessorThreads
      for (i <- processorBeginIndex until processorEndIndex) {
        processors(i) = new Processor(i,time,maxRequestSize,requestChannel,connectionQuotas,connectionsMaxIdleMs,protocol,config.values,metrics)
      }
      
      //Processor线程是附属在Acceptor线程中,随着Acceptor的创建而启动线程
      val acceptor = new Acceptor(endpoint, sendBufferSize, recvBufferSize, brokerId, processors.slice(processorBeginIndex, processorEndIndex), connectionQuotas)
      acceptors.put(endpoint, acceptor)
      
      //启动Acceptor线程
      Utils.newThread("kafka-socket-acceptor-%s-%d".format(protocol.toString, endpoint.port), acceptor, false).start()
      acceptor.awaitStartup()     //等待启动完成,通过CountDownLatch控制,在注册OP_ACCEPT后即可继续
      processorBeginIndex = processorEndIndex
    }
  }
```

在Acceptor中，这个事件就是OP_ACCEPT，表示这个ServerSocketChannel的OP_ACCEPT事件发生了。
因此，Acceptor的accept方法的处理逻辑为：首先通过SelectionKey来拿到对应的ServerSocketChannel，
并调用其accept方法来建立和客户端的连接，然后拿到对应的SocketChannel并交给了processor。
然后Acceptor的任务就完成了，开始去处理下一个客户端的连接请求。

Acceptor只负责接受新的客户端的连接,并将请求转发给Processor处理,采用Round-robin的方式分给不同的Processor

```scala
def run() {
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
    startupComplete()
    var currentProcessor = 0
    
    while (isRunning) {
        val ready = nioSelector.select(500)
        
        if (ready > 0) {
          val keys = nioSelector.selectedKeys()
          val iter = keys.iterator()
          while (iter.hasNext && isRunning) {
              val key = iter.next
              iter.remove()
              if (key.isAcceptable) accept(key, processors(currentProcessor))
              // round robin to the next processor thread
              currentProcessor = (currentProcessor + 1) % processors.length }}}

def accept(key: SelectionKey, processor: Processor) {
    val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
    val socketChannel = serverSocketChannel.accept()
    processor.accept(socketChannel)
}
```

Processor的主要职责是负责从客户端读取数据和将响应返回给客户端，
它本身不处理具体的业务逻辑，也就是说它并不认识它从客户端读取回来的数据。
每个Processor都有一个Selector，用来监听多个客户端，因此可以非阻塞地处理多个客户端的读写请求。

Processor接受一个新的SocketChannel通道连接时,先放入LinkedQueue队列中,然后唤醒Selector线程开始工作
Processor在运行时会首先从通道队列中去取SocketChannel,将客户端连接ID注册到Selector中,
便于后面Selector能够根据ConnectionID获取注册的不同的SocketChannel(比如selector.completedReceives).

```scala
//Acceptor会把多个客户端的数据连接SocketChannel分配一个Processor，因此每个Processor内部都有一个队列来保存这些新来的数据连接
//把一个SocketChannel放到队列中，然后唤醒Processor的selector
def accept(socketChannel: SocketChannel) {
  newConnections.add(socketChannel)
  wakeup()
}


//如果有队列中有新的SocketChannel，则它首先将其OP_READ事件注册到该Processor的selector上面
private def configureNewConnections() {
  while(!newConnections.isEmpty) {
      val channel = newConnections.poll()
      val localHost = channel.socket().getLocalAddress.getHostAddress
      val localPort = channel.socket().getLocalPort
      val remoteHost = channel.socket().getInetAddress.getHostAddress
      val remotePort = channel.socket().getPort
      val connectionId = ConnectionId(localHost, localPort, remoteHost, remotePort).toString
      selector.register(connectionId, channel)
  }
}
```

![](/images/posts/kafka/processor_internal.png)

```scala
val startSelectTime = SystemTime.milliseconds
      val ready = selector.select(300)
      trace("Processor id " + id + " selection time = " + (SystemTime.milliseconds - startSelectTime) + " ms")
      if(ready > 0) {
        val keys = selector.selectedKeys()
        val iter = keys.iterator()
        while(iter.hasNext && isRunning) {
          var key: SelectionKey = null
          try {
            key = iter.next
            iter.remove()
            if(key.isReadable)
              read(key)
            else if(key.isWritable)
              write(key)
            else if(!key.isValid)
              close(key)
            else
              throw new IllegalStateException("Unrecognized key state for processor thread.")

  def read(key: SelectionKey) {
    val socketChannel = channelFor(key)
    var receive = key.attachment.asInstanceOf[Receive]

    if(key.attachment == null) {
      receive = new BoundedByteBufferReceive(maxRequestSize)
      key.attach(receive)
    }

    val read = receive.readFrom(socketChannel)
    val address = socketChannel.socket.getRemoteSocketAddress();
    trace(read + " bytes read from " + address)

    if(read < 0) { close(key)
    } else if(receive.complete) {

      val req = RequestChannel.Request(processor = id, requestKey = key,
        buffer = receive.buffer, startTimeMs = time.milliseconds, remoteAddress = address)

      // 这个是构造函数的参数
      requestChannel.sendRequest(req)

      key.attach(null)
      // explicitly reset interest ops to not READ, no need to wake up the selector just yet
      key.interestOps(key.interestOps & (~SelectionKey.OP_READ))
    } else {
      // more reading to be done
      trace("Did not finish reading, registering for read again on connection " + socketChannel.socket.getRemoteSocketAddress())
      key.interestOps(SelectionKey.OP_READ)
      wakeup()
    }
  }
```

NetworkClient发送Send请求给服务端加入到inFlightRequests,这里服务端发送Send请求给客户端加入到inflightResponses.

问题: 在selector.poll之后为什么要有completedReceives和completedSends的处理逻辑?

解释1: poll操作只是轮询,把注册在SocketChannel上的读写事件获取出来, 那么得到事件后需要进行实际的处理才有用.
比如服务端注册了OP_READ,轮询时读取到客户端发送的请求,那么怎么服务端怎么处理客户端请求呢?就交给了completedReceives
可以把poll操作看做是服务端允许接收这个事件,然后还要把这个允许的事件拿出来进行真正的逻辑处理.

解释2: poll操作并不一定一次性完整地读取完客户端发送的请求,只有等到完整地读取完一个请求,才会出现在completeReceives中!

### RequestChannel

RequestChannel是Processor和Handler交换数据的地方(Processor获取数据后通过RC交给Handler处理)。
它包含了一个队列requestQueue用来存放Processor加入的Request，Handler会从里面取出Request来处理；
它还为每个Processor开辟了一个respondQueue，用来存放Handler处理了Request后给客户端的Response

RequestChannel是SocketServer全局的,一个服务端只有一个RequestChannel.

第一个参数Processor数量用于response,第二个参数队列大小用于request阻塞队列.

```scala
val requestChannel = new RequestChannel(totalProcessorThreads, maxQueuedRequests)

requestChannel.addResponseListener(id => processors(id).wakeup())
```

每个Processor都有一个response队列,而Request请求则是全局的.
为什么request不需要给每个Processor都配备一个队列, 而response则需要呢?

```scala
class RequestChannel(val numProcessors: Int, val queueSize: Int) extends KafkaMetricsGroup {
  private var responseListeners: List[(Int) => Unit] = Nil
  private val requestQueue = new ArrayBlockingQueue[RequestChannel.Request](queueSize)

  private val responseQueues = new Array[BlockingQueue[RequestChannel.Response]](numProcessors)
  for(i <- 0 until numProcessors)
    responseQueues(i) = new LinkedBlockingQueue[RequestChannel.Response]()

  // Send a request to be handled, potentially blocking until there is room in the queue for the request
  // 如果requestQueue满的话，这个方法会阻塞在这里直到有Handler取走一个Request
  def sendRequest(request: RequestChannel.Request) {
    requestQueue.put(request)
  }

  // Send a response back to the socket server to be sent over the network
  def sendResponse(response: RequestChannel.Response) {
    responseQueues(response.processor).put(response)
    for(onResponse <- responseListeners) onResponse(response.processor)
  }

  // Get the next request or block until specified time has elapsed
  // Handler从requestQueue中取出Request，如果队列为空，这个方法会阻塞在这里直到有Processor加入新的Request
  def receiveRequest(timeout: Long): RequestChannel.Request = requestQueue.poll(timeout, TimeUnit.MILLISECONDS)  

  // Get a response for the given processor if there is one
  def receiveResponse(processor: Int): RequestChannel.Response = responseQueues(processor).poll()
}

//object RequestChannel的case class有: Session,Request,Response, ResponseAction.
```

Handler的职责是从requestChannel中的requestQueue取出Request，
处理以后再将Response添加到requestChannel中的responseQueue中。
Processor和Handler互相通信都通过RequestChannel的请求或响应队列

![](/images/posts/kafka/process_handler.png)

现在我们终于可以理清客户端发送请求到服务端处理请求的路径了:

```
NetworkClient --- ClientRequest(Send) --- KafkaChannel ===> SocketServer --- Processor -- RequestChannel -- KafkaRequestHandler
            selector                                                                selector
```

### Request

请求转发给KafkaApis处理, 请求内容都在ByteBuffer buffer中. requestId表示请求类型,有PRODUCE,FETCH等.
提前定义keyToNameAndDeserializerMap,根据requestId,再传入buffer,就可以得到请求类型对应的Request

```scala
  case class Request(processor: Int, connectionId: String, session: Session, private var buffer: ByteBuffer, startTimeMs: Long, securityProtocol: SecurityProtocol) {
    //首先获取请求类型对应的ID
    val requestId = buffer.getShort()

    // NOTE: this map only includes the server-side request/response handlers. Newer request types should only use the client-side versions which are parsed with o.a.k.common.requests.AbstractRequest.getRequest()
    private val keyToNameAndDeserializerMap: Map[Short, (ByteBuffer) => RequestOrResponse]=
      Map(ApiKeys.PRODUCE.id -> ProducerRequest.readFrom, ApiKeys.FETCH.id -> FetchRequest.readFrom)

    //通过ByteBuffer构造请求对象,比如ProducerRequest
    val requestObj = keyToNameAndDeserializerMap.get(requestId).map(readFrom => readFrom(buffer)).orNull

    //请求头和请求内容
    val header: RequestHeader = if (requestObj == null) {buffer.rewind; RequestHeader.parse(buffer)} else null
    val body: AbstractRequest = if (requestObj == null) AbstractRequest.getRequest(header.apiKey, header.apiVersion, buffer) else null

    //重置ByteBuffer为空  
    buffer = null
}
```

KafkaServer 会创建 KafkaRequestHandlerPool, 在 HandlerPool 中会启动numThreads个KafkaRequestHandler线程

KafkaRequestHandler线程从requestChannel的requestQueue中获取Request请求,交给KafkaApis的handle处理

```scala
class KafkaRequestHandler(id: Int, brokerId: Int, val requestChannel: RequestChannel, apis: KafkaApis) extends Runnable

def run() {
  while(true) {
      var req : RequestChannel.Request = null
      while (req == null) {
        req = requestChannel.receiveRequest(300)
      }
      apis.handle(req)
  }
}
```

注意RequestChannel是怎样从KafkaServer一直传递给KafkaRequestHandler的

```scala
class KafkaServer {
  def startup() {
    socketServer = new SocketServer(config, metrics, kafkaMetricsTime)
    socketServer.startup()

    apis = new KafkaApis(socketServer.requestChannel, replicaManager, consumerCoordinator, ...)
    requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, 
        config.numIoThreads)
  }
}

class KafkaRequestHandlerPool(val brokerId: Int, val requestChannel: RequestChannel, 
    val apis: KafkaApis, numThreads: Int) {
  
  for(i <- 0 until numThreads) {
    runnables(i) = new KafkaRequestHandler(i, brokerId, aggregateIdleMeter, numThreads, requestChannel, apis)
    threads(i) = Utils.daemonThread("kafka-request-handler-" + i, runnables(i))
    threads(i).start()
  }
}

class KafkaRequestHandler(id: Int, brokerId: Int, val requestChannel: RequestChannel, apis: KafkaApis) {}
```

![](/images/posts/kafka/socker_server_handler.png)


Ref:

[l1](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Internals)
[l2](https://zqhxuyuan.github.io/2016/01/08/2016-01-08-Kafka_SocketServer/)
[l3](http://colobu.com/2015/03/23/kafka-internals/)