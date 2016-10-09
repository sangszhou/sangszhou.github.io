---
layout: post
title: Java NIO Netty
categories: [nio]
keywords: nio, java
---

Java Nio 和 Netty 例子与原理

有很强的套路性, 但是容易忘

## NIO core

### SelectionKey

SelectionKey是表示一个Channel和Selector的注册关系。在Acceptor中的selector，
只有监听客户端连接请求的ServerSocketChannel的OP_ACCEPT事件注册在上面。
当selector的select方法返回时，则表示注册在它上面的Channel发生了对应的事件。

SelectionKey 中到底放了多少东西?

```java
public abstract class SelectionKey {
    public abstract SelectableChannel channel();
    public abstract Selector selector();
    public abstract int interestOps();
    public abstract SelectionKey interestOps(int ops); // 感兴趣的事件
    private volatile Object attachment = null;
    public final Object attach(Object ob) return attachmentUpdater.getAndSet(this, ob);
```

SelectionKey 的常用操作

```scala
key.interestOps(SelectionKey.OP_READ) // 添加兴趣事件
key.attach(curr) // 添加 SelectionAttachment, 它的类型是 Object

def write(key: SelectionKey) {
    val socketChannel = key.channel().asInstanceOf[SocketChannel]
    val response = key.attachment().asInstanceOf[RequestChannel.Response]
    val responseSend = response.responseSend
    // responseSend 内部会记录已经发送的数据长度
    val written = responseSend.writeTo(socketChannel)
        // var written = channel.write(Array(sizeBuffer, buffer))
    if(responseSend.complete) {
        key.attach(null)
        key.interestOps(SelectionKey.OP_READ)
    } else {
        key.interestOps(SelectionKey.OP_WRITE)
        wakeup()
    }

def read(key: SelectionKey) {
    val socketChannel = key.channel().asInstanceOf[SocketChannel]
    // client 发送的部分数据暂存在 channel 上
    var receive = key.attachment.asInstanceOf[Receive]
    if(key.attachment == null) {
        receive = new BoundedByteBufferReceive(maxRequestSize)
        key.attach(receive)
    }
    val read = receive.readFrom(socketChannel) // 一次不一定读完
    val address = socketChannel.socket.getRemoteSocketAddress();
    trace(read + " bytes read from " + address)
    if(read < 0) { close(key)
    } else if(receive.complete) {
        val req = RequestChannel.Request(processor = id, requestKey = key,
            buffer = receive.buffer, startTimeMs = time.milliseconds, remoteAddress = address)
            
        requestChannel.sendRequest(req) // 放到公共的 BlockingQueue 中, 待处理
        key.attach(null)
        key.interestOps(key.interestOps & (~SelectionKey.OP_READ))
    } else {
        key.interestOps(SelectionKey.OP_READ)
        wakeup()
    }
```

读取数据的过程, 一般来说, 传输协议会定义前几个字节表示待传输数据的长度

对于写数据而言, 待发送数据的大小是已知的, 不需要传输, 并且这个待发送数据的前几个字节应该也是
传输数据的长度

```scala
  def readFrom(channel: ReadableByteChannel): Int = {
    expectIncomplete()
    var read = 0
    // have we read the request size yet?
    if(sizeBuffer.remaining > 0)
      read += Utils.read(channel, sizeBuffer)
    // have we allocated the request buffer yet?
    if(contentBuffer == null && !sizeBuffer.hasRemaining) {
      sizeBuffer.rewind()
      val size = sizeBuffer.getInt()
      if(size <= 0)
        throw new InvalidRequestException("%d is not a valid request size.".format(size))
      if(size > maxSize)
        throw new InvalidRequestException("Request of length %d is not valid, it is larger than the 
            maximum size of %d bytes.".format(size, maxSize))
            
      contentBuffer = byteBufferAllocate(size)
    }
    // if we have a buffer read some stuff into it
    if(contentBuffer != null) {
      read = Utils.read(channel, contentBuffer)
      // did we get everything?
      if(!contentBuffer.hasRemaining) {
        contentBuffer.rewind()
        complete = true
      }
    }
    read
  }
```

### Buffer

![](/images/posts/kafka/NIOBuffer.png)

read write mode 的区别在于，read mode 的 limit 是 write mode 的 position

When you write data into a buffer, the buffer keeps track of how much data you have written. 
Once you need to read the data, you need to switch the buffer from writing mode into reading 
mode using the flip() method call. In reading mode the buffer lets you read all the data written into the buffer. 

```java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;}
```

Once you have read all the data, you need to clear the buffer, to make it ready for writing again. 
You can do this in two ways: By calling clear() or by calling compact(). The clear() method 
clears the whole buffer. The compact() method only clears the data which you have already read. 
Any unread data is moved to the beginning of the buffer, and data will now be written into 
the buffer after the unread data.

**rewind**

The Buffer.rewind() sets the position back to 0, so you can reread all the 
data in the buffer. The limit remains untouched, thus still marking how many 
elements (bytes, chars etc.) that can be read from theBuffer.

```java
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;}
```

