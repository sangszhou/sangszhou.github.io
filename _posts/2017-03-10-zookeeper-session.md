---
layout: post
title:  "Zookeeper code part 4 session, client server interact"
date:   "2016-03-10 17:50:00"
categories: zookeeper
keywords: zookeeper
---

The 4th post about zookeeper, more post will come...

### Expiry queue

存放 Session 和 Connection 的超时数据结构叫做 ExpiryQueue, 它的实现也蛮简单，就是一个带有 bucket 的 Map, 举个例子，假设现在 zk 上有几个 session

```
1 100
2 121
3 123
4 158
```

第一列是 session, 第二列是 timeout 时间，在 expiryQueue 中，timeout 时间的记录并不是那么精确，会进行 round

```java
private long roundToNextInterval(long time) {
    return (time / expirationInterval + 1) *  expirationInterval;
}

// The maximum number of buckets is equal to max timeout/expirationInterval,
// so the expirationInterval should not be too small compared to the
// max timeout that this expiry queue needs to maintain.
```

和 kakfa 的 timingwheel 相比，这里简单些，要注意，expirationInterval 的时间可能太短，太短的话桶就太多了

所以，上面的 session 和 timeout 可能会被 round 成这样

```
1 120
2 140
3 140
4 160
```

体现在代码中的数据结构就是

```java
private final ConcurrentHashMap<E, Long> elemMap = new ConcurrentHashMap<E, Long>();
private final ConcurrentHashMap<Long, Set<E>> expiryMap = new ConcurrentHashMap<Long, Set<E>>();

eleMap
    1 120
    2 140
    3 140
    4 160
expiryMap
    120 <1>
    140 <2, 3>
    160 <4>    
```

### SessionImpl

```java
public static interface Session {
    long getSessionId();
    int getTimeout();
    boolean isClosing();
    }
```


### client start 

所有的 client 在开始时都必须启动 zookeeper Instance.

This is the main class of ZooKeeper client library. To use a ZooKeeper
  service, an application must first instantiate an object of ZooKeeper class.
 All the iterations will be done by calling the methods of ZooKeeper class.
 The methods of this class are thread-safe unless otherwise noted.
 
Once a connection to a server is established, a session ID is assigned to the
client. The client will send heart beats to the server periodically to keep
the session valid.

The application can call ZooKeeper APIs through a client as long as the
session ID of the client remains valid.

If for some reason, the client fails to send heart beats to the server for a
 prolonged period of time (exceeding the sessionTimeout value, for instance),
 the server will expire the session, and the session ID will become invalid.
 The client object will no longer be usable. To make ZooKeeper API calls, the
 application must create a new client object.  !!! [也就是说要重新 new 一个么]

 If the ZooKeeper server the client currently connects to fails or otherwise
 does not respond, the client will automatically try to **connect to another
 server before its session ID expires**. If successful, the application can
 continue to use the client. [这个就是 global session 的原理么]
 这个 server 是怎么知道这个 session 还有效呢？

 The ZooKeeper API methods are either **synchronous or asynchronous**. Synchronous
 methods blocks until the server has responded. Asynchronous methods just queue
 the request for sending and return immediately. They take a callback object that
 will be executed either on successful execution of the request or on error with
 an appropriate return code (rc) indicating the error.
 
 Some successful ZooKeeper API calls can leave watches on the "data nodes" in
 the ZooKeeper server. Other successful ZooKeeper API calls can trigger those
 watches. Once a watch is triggered, an event will be delivered to the client
 which left the watch at the first place. **Each watch can be triggered only
 once.** Thus, up to one event will be delivered to a client for every watch it
 leaves.

 A client needs an object of a class implementing Watcher interface for
 processing the events delivered to the client.

 When a client drops current connection and re-connects to a server, all the
 existing watches are considered as being triggered but the **undelivered events
 are** lost. To emulate this, the client will generate a special event to tell
 the event handler a connection has been dropped. This special event has type
 EventNone and state sKeeperStateDisconnected.

```java
public class ZooKeeper implements AutoCloseable {
    protected final ClientCnxn cnxn;
    // 可用 server, "host1:port1,host2:port2
    protected final HostProvider hostProvider;
    protected final ZKWatchManager watchManager;
    private final ZKClientConfig clientConfig;
    
    public enum States { // 允许的状态
            CONNECTING, ASSOCIATING, CONNECTED, CONNECTEDREADONLY,
            CLOSED, AUTH_FAILED, NOT_CONNECTED;
    }
    
    public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
                     boolean canBeReadOnly, HostProvider aHostProvider,
                     ZKClientConfig clientConfig) throws IOException {
        
        cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
        cnxn.start();    
    }    
    // 这个好像是 server 生成的把
    public byte[] getSessionPasswd() {
            return cnxn.getSessionPasswd();
    }
    
    // default watcher
    public synchronized void register(Watcher watcher) {
            watchManager.defaultWatcher = watcher;
    }
    
    // async version
    public void create(final String path, byte data[], List<ACL> acl, CreateMode createMode, Create2Callback cb, Object ctx, long ttl) {
        final String clientPath = path;
        PathUtils.validatePath(clientPath, createMode.isSequential());
        EphemeralType.validateTTL(createMode, ttl);

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        setCreateHeader(createMode, h);
        ReplyHeader r = new ReplyHeader();
        Create2Response response = new Create2Response();
        Record record = makeCreateRecord(createMode, serverPath, data, acl, ttl);
        cnxn.queuePacket(h, r, record, response, cb, clientPath, serverPath, ctx, null);
    }    
    
    public String create(final String path, byte data[], List<ACL> acl,
                         CreateMode createMode) throws KeeperException, InterruptedException {
        final String clientPath = path;
        PathUtils.validatePath(clientPath, createMode.isSequential());
        EphemeralType.validateTTL(createMode, -1);

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        h.setType(createMode.isContainer() ? ZooDefs.OpCode.createContainer : ZooDefs.OpCode.create);

        CreateRequest request = new CreateRequest();
        CreateResponse response = new CreateResponse();
        request.setData(data);
        request.setFlags(createMode.toFlag());
        request.setPath(serverPath);

        if (acl != null && acl.size() == 0) {
            throw new KeeperException.InvalidACLException();
        }
        request.setAcl(acl);
        ReplyHeader r = cnxn.submitRequest(h, request, response, null);

        if (r.getErr() != 0) {
            throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                    clientPath);
        }
        // 这是个 blocking 操作，is not ready, wait()
        if (cnxn.chrootPath == null) {
            return response.getPath();
        } else {
            return response.getPath().substring(cnxn.chrootPath.length());
        }
    }
    
}
```