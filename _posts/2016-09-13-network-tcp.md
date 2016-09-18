---
layout: post
title: TCP 网络
categories: [networking]
keywords: networking
---

尽管TCP和UDP都使用相同的网络层（IP），TCP却向应用层提供与UDP完全不同的服务。
TCP提供一种面向连接的、可靠的字节流服务。

面向连接意味着两个使用TCP的应用（通常是一个客户和一个服务器）在彼此交换数据之前必须先建立一个TCP连接。
这一过程与打电话很相似，先拨号振铃，等待对方摘机说“喂”，然后才说明是谁。


## TCP 报文格式

![](/images/posts/network/tcp-package.png)

有几个字段比较重要

>  序号: Seq 序号, 占 32 位, 用来标识从 TCP 源端向目的端发送的字节流, 发起方发送数据时对此进行标记
>  确认序号, Ack 序号, 占 32 位, 只有 ACK 标识位为 1 时, 确认序号字段才有效, Ack = Seq + 1
>  标志位: 共 6 位, 包括 ACK, RST(重置链接), SYN(发起连接), FIN(释放链接), PSH(接收方应该尽快把包交到上层), URG(紧急)

## 三次握手协议
所谓三次握手（Three-Way Handshake）即建立TCP连接，就是指建立一个TCP连接时，需要客户端和服务端
总共发送3个包以确认连接的建立。在socket编程中，这一过程由客户端执行connect来触发，整个流程如下图所示：

![](/images/posts/network/handshake.png)

**第一次握手**

client 将标志位 SYN 置为 1, 随机产生一个值 Seq = j, 并将数据包发送到 Server, 
Client 进入 SYN_SENT 状态, 等待 Server 确认

**第二次握手**

Server 收到数据包后由标志位 SYN = 1 知道 Client 请求连接, Server 将标志位 SYN 和
置为 1, ack = j+1, 随机产生一个值 Seq = k, 并将该数据包发送 Client 端, server 进入
SYN_RCVD 状态

**第三次握手**

Client 收到确认后, 检查 ACK 是否为 j+1 ack 是否为 1, 如果正确则将标志位 ACK 置为 1, ack = k+1, 并将数据包
发送给 server, server 检查 ack 是否为 k+1, ack 是否为 1, 如果正确连接则建立连接, client & server 进入到
ESTABLISHED 状态, 完成三次握手, 随后 client & server 之间可以开始传输数据

**SYN 攻击** 

client 伪造大量不存在的 IP 地址, 并向 server 不断的发送 SYN 包, server 回复确认包, server 不断重发直至超时,
这些伪造的 SYN 包将长时间占用未连接队列, 导致正常的 SYN 请求因为队列满而被丢弃, 从而引起网络堵塞甚至瘫痪。 SYN
攻击是典型的 DDOS 攻击

```
netstat -nap | grep SYN_RCVD
```

如果有大量的半连接状态且源地址是随机的, 则肯定遭到了 SYN 攻击

## 四次挥手

三次握手耳熟能详，四次挥手估计就少有人知道了。所谓四次挥手（Four-Way Wavehand）即终止TCP连接，就是指
断开一个TCP连接时，需要客户端和服务端总共发送4个包以确认连接的断开。在socket编程中，这一过程由客户端或服务
端任一方执行close来触发，整个流程如下图所示：

![](/images/posts/network/wavehand.png)

由于 TCP 连接时全双工的，因此，每个方向都必须要单独进行关闭，这一原则是当一方完成
数据发送任务后，发送一个 FIN 来终止这一方向的连接，收到一个 FIN 只是意味着这一方
向上没有数据流动了，即不会再收到数据了，但是在这个 TCP 连接上仍然能够发送数据，
直到这一方向也发送了 FIN 。首先进行关闭的一方将执行主动关闭，而另一方则执行被动关闭，上
图描述的即是如此

**第一次挥手:**
Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态

**第二次挥手:**
Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态

**第三次挥手:**
Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态

**第四次挥手:**
client 收到 FIN 后，client 进入 TIME_WAIT 状态，接着发送一个ACK给Server，确认序号为收到
序号+1，Server进入CLOSED状态，完成四次挥手

很多时候, Linux 机器的端口处在 TIME_WAIT 状态, 这是因为 TIME_WAIT 状态持续的时间太长了, 应该尽量少些

但是这是 client 的状态, 和 server 没什么关系

上面是一方主动关闭，另一方被动关闭的情况，实际中还会出现同时发起主动关闭的情况，具体流程如下图：

![](/images/posts/network/wavehandtogether.png)

### 为什么建立连接是三次握手，而关闭连接却是四次挥手呢?

这是因为服务端在LISTEN状态下，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里发送给客户端。
而关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部
数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表
示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送。

简单的说就是存在半断开状态

## VPN, DHCP
