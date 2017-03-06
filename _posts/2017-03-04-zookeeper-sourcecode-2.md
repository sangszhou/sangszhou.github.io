Processor 处理链

### Resource

1. http://www.myexception.cn/operating-system/1813820.html 带图的分析，图可以参考

### FAQ

1. Session 是怎么在多态机器上共享的？还是压根不共享，一个 client 只练一个 server, 在这个 server 上验证信息？


### Fast Path

Leader 和 follower 有不同的责任链构成。Leader 较为复杂，有 prepRequestProcessor, ProposalProcessor, CommitProcessor 和 FinalProcessor, 在 ProposalProcessor 后面还有 SyncProcessor 和 AckProcessor. 它们各自的功能有：prepRequestProcessor 用于接受请求，按照请求的种类分别处理，对于写请求，把 Request 变成 ChangeRecord 发送到 ProposalProcessor, 对于读请求，发到哪里去？

ProposalProcessor 做两件事，第一件事是把消息发送给所有的 follower 投票，发送逻辑是两阶段提交，第一阶段把请求发送到 follower, follower 和自身对此消息进行 Ack 并记录到 log 中。当 leader 收到足够的 ack 以后 tryToCommit, 把消息再发送给所有的 follower 和 observer. 其中 SyncProcessor 用来记录 log，AckProcessor 就更简单了，它是 Leader 自己投票给自己，直接通过共享内存的方式，而不是通过 socket 发一个消息给自己。CommitProcessor 是干嘛的？

与 Leader 相比，Follower 就简单的多了，它主要用来处理读请求，对于写请求，直接转发给 Leader. 它有三个 Processor, 分别是 FollowerRequestProcessor，识别是否是事务，如果是事务请求，直接转发给 Leader, SendAckProcessor 用来干嘛的？还有没有其他的 Processor ？

除了处理链以外，承载处理链的载体，ZKServer 也很重要，它有一个较为复杂的层次结构，每个层次结构处理对应的逻辑，多个层次结构之间的逻辑倒并不是很清晰。

```
ZookeeperServer
  QuorumZookeeperServer
    LearnerZookeeperServer
      FollowerZookeeperServer
      ObserverZookeeperServer
  LeaderZookeeperServer
```

## Details


### ZooKeeperServer

```java
public class ZooKeeperServer implements SessionExpirer, ServerStats.Provider {
  protected SessionTracker sessionTracker;
  private ZKDatabase zkDb;
  private FileTxnSnapLog txnLogFactory = null;
  private final AtomicLong hzxid = new AtomicLong(0);
  protected RequestProcessor firstProcessor;

  final List<ChangeRecord> outstandingChanges = new ArrayList<ChangeRecord>();
  // this data structure must be accessed under the outstandingChanges lock
  final HashMap<String, ChangeRecord> outstandingChangesForPath = new HashMap<String, ChangeRecord>();

  protected ServerCnxnFactory serverCnxnFactory;
  private final ZooKeeperServerListener listener;

  public synchronized void startup() {
    if (sessionTracker == null) {
        createSessionTracker();
    }
    startSessionTracker();
    setupRequestProcessors();

    registerJMX();

    setState(State.RUNNING);
    notifyAll();
  }

// 为什么默认是这三个链子？
  protected void setupRequestProcessors() {
      这个方法在子类中都被重写了，所以它本身的内容并不重要
  }

  long createSession(ServerCnxn cnxn, byte passwd[], int timeout) {

      long sessionId = sessionTracker.createSession(timeout);
      Random r = new Random(sessionId ^ superSecret);
      r.nextBytes(passwd);
      ByteBuffer to = ByteBuffer.allocate(4);
      to.putInt(timeout);
      cnxn.setSessionId(sessionId); // session 保存在 cnxn 中

      // 这是要发给谁？
      Request si = new Request(cnxn, sessionId, 0, OpCode.createSession, to, null);
      setLocalSessionFlag(si);
      submitRequest(si);
      return sessionId;
  }  
}


```

### session

在接收到消息后，会判断是否已经初始化完毕了，如果没初始化，先创建 Session

```java
//class NettyServerCnxn
public void receiveMessage(ChannelBuffer message) {
  if (initialized) {
      zks.processPacket(this, bb);

      if (zks.shouldThrottle(outstandingCount.incrementAndGet())) {
          disableRecvNoWait();
      }
  } else {
      LOG.debug("got conn req request from " + getRemoteSocketAddress());
      zks.processConnectRequest(this, bb);
      initialized = true;
  }
}

// 这个也是在 zookeeperServer 第一层来做处理的
// 如果有 session 了再发 connect, 就会 reopen session
public void processConnectRequest(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    BinaryInputArchive bia = BinaryInputArchive.getArchive(new ByteBufferInputStream(incomingBuffer));
    ConnectRequest connReq = new ConnectRequest();
    connReq.deserialize(bia, "connect");
    byte passwd[] = connReq.getPasswd(); // 还要传递密码的
    cnxn.setSessionTimeout(sessionTimeout);
    long sessionId = connReq.getSessionId();
    if (sessionId == 0) {
      createSession(cnxn, passwd, sessionTimeout);
    } else {
      long clientSessionId = connReq.getSessionId();
      cnxn.setSessionId(sessionId);
      reopenSession(cnxn, sessionId, passwd, sessionTimeout);
    }    
}
```


SessionTracker 有很多子类

```java
public interface SessionTracker {
  long createSession(int sessionTimeout);
  boolean addGlobalSession(long id, int to);
  boolean addSession(long id, int to);
  boolean touchSession(long sessionId, int sessionTimeout);
}

public abstract class UpgradeableSessionTracker implements SessionTracker {
  private ConcurrentMap<Long, Integer> localSessionsWithTimeouts;
  protected LocalSessionTracker localSessionTracker;
  /**
   * Upgrades the session to a global session.
   * This simply removes the session from the local tracker and marks
   * it as global.  It is up to the caller to actually
   * queue up a transaction for the session.
   *
   * @param sessionId
   * @return session timeout (-1 if not a local session)
   */
  public int upgradeSession(long sessionId) {
      if (localSessionsWithTimeouts == null) {
          return -1;
      }
      // We won't race another upgrade attempt because only one thread
      // will get the timeout from the map
      Integer timeout = localSessionsWithTimeouts.remove(sessionId);
      if (timeout != null) {
          LOG.info("Upgrading session 0x" + Long.toHexString(sessionId));
          // Add as global before removing as local
          addGlobalSession(sessionId, timeout);
          localSessionTracker.removeSession(sessionId);
          return timeout;
      }
      return -1;
  }
}

public class LeaderSessionTracker extends UpgradeableSessionTracker {
  private final boolean localSessionsEnabled;

  private final SessionTrackerImpl globalSessionTracker;
  public boolean addGlobalSession(long sessionId, int sessionTimeout) {
      boolean added = globalSessionTracker.addSession(sessionId, sessionTimeout);
      if (localSessionsEnabled && added) {
          // Only do extra logging so we know what kind of session this is
          // if we're supporting both kinds of sessions
          LOG.info("Adding global session 0x" + Long.toHexString(sessionId));
      }
      return added;
  }  
}

/**
 * This is a full featured SessionTracker. It tracks session in grouped by tick
 * interval. It always rounds up the tick interval to provide a sort of grace
 * period. Sessions are thus expired in batches made up of sessions that expire
 * in a given interval.
 */
public class SessionTrackerImpl extends ZooKeeperCriticalThread implements SessionTracker {
  protected final ConcurrentHashMap<Long, SessionImpl> sessionsById = new ConcurrentHashMap<Long, SessionImpl>();  
  private final ExpiryQueue<SessionImpl> sessionExpiryQueue;
  private final ConcurrentMap<Long, Integer> sessionsWithTimeout;
  private final AtomicLong nextSessionId = new AtomicLong();

  @Override
  public void run() {
      try {
          while (running) {
              long waitTime = sessionExpiryQueue.getWaitTime();
              if (waitTime > 0) {
                  Thread.sleep(waitTime);
                  continue;
              }

              for (SessionImpl s : sessionExpiryQueue.poll()) {
                  setSessionClosing(s.sessionId);
                  expirer.expire(s);
              }
          }
      } catch (InterruptedException e) {
          handleException(this.getName(), e);
      }
      LOG.info("SessionTrackerImpl exited loop!");
  }

}

```

http://blog.csdn.net/jeff_fangji/article/details/44122541 不错的资料
http://blog.csdn.net/jeff_fangji/article/details/41786205

### Leader Processor
