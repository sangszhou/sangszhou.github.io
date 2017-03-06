应该会有 3 篇 blog 分析 zookeeper 的源代码，这是第一部分，分析 Leader/follower 的启动和处理流程，分析的版本是 3.6

## 服务器启动

服务器启动有两种方式，第一种是 stand alone 版本的启动，第二种是 cluster 模式。我不太了解为什么这两种启动方式会不同，我觉得封装好的话，启动方式应该一直才对。

两种启动方式的入口都是 QuorumPeerMain.main

```java
protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
    // Start and schedule the the purge task
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
            .getDataDir(), config.getDataLogDir(), config
            .getSnapRetainCount(), config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.isDistributed()) {
        runFromConfig(config);
    } else { // 启动单机版
        ZooKeeperServerMain.main(args);
    }
}
```

### 先来分析 cluster 版本的启动

cluster 版本下有三种角色，分别是 leader, follower 和 observer. f/o 角色的作用和逻辑类似，不再详细些 observer 的启动过程。Networking 就只看 netty 版本的。和想象中的一样，系统会先启动 NettyServerCnxnFactory, 它是 nettyServer, 负责接收数据，接收到的数据传递到上层供处理（还需要更加详细的内容）, 从 netty server 的数据到 leader/follower 的过程现在也没搞清楚，所以先 pending. 当 消息到达 leader 的 lead 方法时，leader 会采取对应的处理，具体的方法是 lead, 而 follower 的主要逻辑在 followerLeader 中。对于过来的请求，要么转发给责任链来处理，要么自己直接就在 Leader/follower 中处理。

```java
public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {
  cnxnFactory = ServerCnxnFactory.createFactory(); // 根据配置文件反射出 cnxnFactory
  quorumPeer = getQuorumPeer(); // 创建 Peer 接下来是各种 ServerCnxnFactory
  quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
  quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
  quorumPeer.setCnxnFactory(cnxnFactory);
  quorumPeer.start();
  quorumPeer.join();
}
```

QuorumPeer 是 Thread 的子类，它的启动过程就是加载数据库，启动 connection 并开始选举 Leader

```java
public synchronized void start() {
  loadDataBase();
  startServerCnxnFactory();
  startLeaderElection();
  super.start();
}
```

最后一行很关键，它调用默认的 start 方法，其实是启动线程的 run 逻辑，开始处理。注意，startLeaderElection 并没有开支执行 leader 选举，知识把当前节点的状态设置为 LOOKING 而已。 loadDataBase 不再赘述了，就是初始化自身的信息。QuorumPeer 的成员变量特别多

```java
// 自身的状态，只能是他们中的一个
  public Follower follower;
  public Leader leader;
  public Observer observer;

  QuorumCnxnManager mgr // 它仅负责 FastLeaderElection 与其他节点通信
  ZKDatabase zkDb;
  QuorumVerifier // 不知道他的意义
  long myid
// 难道是 3 个端口？
  InetSocketAddress myQuorumAddr, myElectionAddr, myClientAddir

// This is who I think the leader currently is.
  Vote currentVote
  Election electionAlg;  // 选举算法
  ServerCnxnFactory cnxnFactory; // 连接器，server 之间通信的
  private FileTxnSnapLog logFactory = null;
  AdminServer adminServer;
```

先说端口的问题，client 端口一般是 2181

```
# The number of milliseconds of each tick
tickTime=2000

# The number of ticks that the initial
# synchronization phase can take
initLimit=10 // 会等这么久才算开始

# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5

# the directory where the snapshot is stored.
dataDir=/usr/local/zk/data

# the port at which the clients will connect
clientPort=2183

#the location of the log file
dataLogDir=/usr/local/zk/log

server.0=hadoop:2288:3388
server.1=hadoop0:2288:3388
server.2=hadoop1:2288:3388

server.A=B：C：D
A：其中 A 是一个数字，表示这个是服务器的编号；
B：是这个服务器的 ip 地址；
C：Leader选举的端口；
D：Zookeeper服务器之间的通信端口。
```

Quorum 的执行过程，主要逻辑

```java
while (running) {
  switch (getPeerState()) {
    case LOOKING:
      ... startLeaderElection();
    case OBSERVING:
        setObserver(makeObserver(logFactory));
        observer.observeLeader();
    case FOLLOWING:
      setFollower(makeFollower(logFactory));
      follower.followLeader();
    case LEADING:
      setLeader(makeLeader(logFactory));
      leader.lead();
      setLeader(null);
  }
}
```

选举完以后，这个 Quorum 就成为了 3 个角色中的一个，开始执行。 Leader.lead, Follower.followerLeader。除了这两个主逻辑外还有很多线程做一些 dedicated 的工作，比如 Leader 中有 socketAcceptor 线程，有发送 packet 线程, LearderHandler 有专门的线程等等

Leader 的代码大约 1400 行，它是最负责的一个系统

```java
void lead() throws IOException, InterruptedException {
  zk.loadData();
  cnxAcceptor = new LearnerCnxAcceptor(); // 接受 socket, 每个 socket 创建一个 LearnerHandler, 是一个线程
  cnxAcceptor.start();
  // wait majority server in sync with leader, 这个方法阻塞等待，但是并不发送数据
  // 发送数据和 follower sync 发生在 learnerHandler 中，有握手协议
  waitForEpochAck(self.getId(), leaderStateSummary)
  // 这行代码不知道在等什么
  waitForNewLeaderAck(self.getId(), zk.getZxid(), LearnerType.PARTICIPANT);

  startZkServer(); // 启动 责任链 processor

  while(true) {
    SyncedLearnerTracker syncedAckSet
    Learners.filter(_.synced).foreach(syncedAckSet.add(_))
    learners.foreach(_.ping)
  }
```

走了一圈，没看到处理逻辑在哪。Leader 和 Follower 通信的地方都在 LearnerHandler 中，它是非常重要的一个线程，Handler 由 acceptor 创建，可用的链接由 SyncedLearnerTracker 维持。当消息到来以后，LearnerHandler 会调用 Leader 和 LeaderZKServer 中的方法来处理之。

LearnerHandler 与 follower 连接上以后，先是握手协议, 然后等待正常的消息。LearnerHandler 是 Leader 的主要执行过程
代码量 1000 行

```java
class LearnerCnxAcceptor extends ZooKeeperCriticalThread {
  public void run() {
    while (!stop) {
        Socket s = ss.accept();
        s.setSoTimeout(self.tickTime * self.initLimit);
        s.setTcpNoDelay(nodelay);
        LearnerHandler fh = new LearnerHandler(s, Leader.this);
        fh.start();
    }
  }
}

class LearderHandler {
  Socket sock
  final Leader leader;
  final LinkedBlockingQueue<QuorumPacket> queuedPackets = new LinkedBlockingQueue<QuorumPacket>();
  private BinaryInputArchive ia;
  private BinaryOutputArchive oa
  private BufferedOutputStream bufferedOutput;

  run() {
      QuorumPacket qp = new QuorumPacket();
      ia.readRecord(qp, "packet");
      // 第一个消息必须是下面两种
      if (qp.getType() != Leader.FOLLOWERINFO && qp.getType() != Leader.OBSERVERINFO) { error }

      // 获得 learner 的 zxid
      long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
      // shakehand, step2, send LearnerInfo to learner
      QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, newLeaderZxid, ver, null);
      oa.writeRecord(newEpochPacket, "packet");

      // wait ack
      ia.readRecord(ackEpochPacket, "packet");
      if (ackEpochPacket.getType() != Leader.ACKEPOCH) { error }
      leader.waitForEpochAck(this.getSid(), ss); // if success, epoch ack add one

      // 非常重要，后面会继续分析
      synFollower()

      // sending new message ${sid} 这个消息的用处是什么
      QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER,
                        newLeaderZxid, leader.self.getLastSeenQuorumVerifier()
                        .toString().getBytes(), null);
      queuedPackets.add(newLeaderQP);

      // 一条线程来发消息，从 queuedPackets 中 poll 并发送
      startSendingPackets(); // 发送消息， 把 queuedPackets 中的消息发出去，第一个消息应该就是 NEWLEADER 吧
      qp = new QuorumPacket();
      ia.readRecord(qp, "packet");
      if (qp.getType() != Leader.ACK) { error }
      // 如果成功，则说明第一个消息已经交互完毕了，这个是否有必要呢？
      leader.waitForNewLeaderAck(getSid(), qp.getZxid(), getLearnerType());

      // Sending UPTODATE message 这个是干啥的？
      queuedPackets.add(new QuorumPacket(Leader.UPTODATE, -1, null, null));

      // 准备工作完成，开始同步消息
      while (true) {
        qp = new QuorumPacket();
        ia.readRecord(qp, "packet");
        switch (qp.getType()) {
          case Leader.REQUEST:
            Request si;
            if (type == OpCode.sync) {
              si = new LearnerSyncRequest(this, sessionId, cxid, type, bb, qp.getAuthinfo());
            } else {
              si = new Request(null, sessionId, cxid, type, bb, qp.getAuthinfo());
            }
            si.setOwner(this); // 为了返回给 follower ?
            leader.zk.submitLearnerRequest(si); // 交给 processor 去处理
          case Leader.ACK:
          // 直接调用 leader 处理，而不用再调用 processor, 代码里面进行 commit/inform 操作
            leader.processAck(this.sid, qp.getZxid(), sock.getLocalSocketAddress());
          case Leader.PING:
            // leader not ping back
          case Leader.REVALIDATE:

        }
      }
  }
}
```

Leader 有几个重要的方法，一个是 syncFollower, processAck 还有一个是 submitLearnerRequest, 虽然他是 processor 的内容

```java
//Determine if we need to sync with follower using DIFF/TRUNC/SNAP
//and setup follower to receive packets from commit processor
public boolean syncFollower(long peerLastZxid, ZKDatabase db, Leader leader) {
  /*
   * Here are the cases that we want to handle
   *
   * 1. Force sending snapshot (for testing purpose)
   * 2. Peer and leader is already sync, send empty diff
   * 3. Follower has txn that we haven't seen. This may be old leader
   *    so we need to send TRUNC. However, if peer has newEpochZxid,
   *    we cannot send TRUC since the follower has no txnlog
   * 4. Follower is within committedLog range or already in-sync.
   *    We may need to send DIFF or TRUNC depending on follower's zxid
   *    We always send empty DIFF if follower is already in-sync
   * 5. Follower missed the committedLog. We will try to use on-disk
   *    txnlog + committedLog to sync with follower. If that fail,
   *    we will send snapshot
   */
   else if (lastProcessedZxid == peerLastZxid) {
     // Follower is already sync with us, send empty diff
    queueOpPacket(Leader.DIFF, peerLastZxid);
  }else if (peerLastZxid > maxCommittedLog && !isPeerNewEpochZxid) {
    // Newer than commitedLog, send trunc and done
    queueOpPacket(Leader.TRUNC, maxCommittedLog);
  } else if ((maxCommittedLog >= peerLastZxid) && (minCommittedLog <= peerLastZxid)) {
    // within range
    Iterator<Proposal> itr = db.getCommittedLog().iterator();
    // Queue committed proposals into packet queue. The range of packets which
    // is going to be queued are (peerLaxtZxid, maxZxid]
    currentZxid = queueCommittedProposals(itr, peerLastZxid, null, maxCommittedLog);
  } else if (peerLastZxid < minCommittedLog && txnLogSyncEnabled) {
    // Use txnlog and committedLog to sync
    // Calculate sizeLimit that we allow to retrieve txnlog from disk
    LOG.info("Use txnlog and committedLog for peer sid: " + getSid());
    LOG.debug("Queueing committedLog 0x" + Long.toHexString(currentZxid));
  }
}


synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {
// 找到对应的 proposal
  Proposal p = outstandingProposals.get(zxid);
  if (p == null) {error}
  boolean hasCommitted = tryToCommit(p, zxid, followerAddr);
}

synchronized public boolean tryToCommit(Proposal p, long zxid, SocketAddress followerAddr) {
  outstandingProposals.remove(zxid);
  if (p.request != null) {
    toBeApplied.add(p);
  }
  // commit 发送失败也没关系的吗
  commit(zxid);
  inform(p);
  zk.commitProcessor.commit(p.request);

// 这是在干嘛
  if (pendingSyncs.containsKey(zxid)) {
    for (LearnerSyncRequest r : pendingSyncs.remove(zxid)) {
      sendSync(r);
    }
  }
}
// 这部分内容到第二篇文章中说明
public void submitLearnerRequest(Request request) {
  prepRequestProcessor.processRequest(request);
}
```
Leader 的逻辑分析到此结束

### Follower 的处理逻辑

Follower 的处理逻辑主要在 followLeader 中

```java
void followLeader() throws InterruptedException {
  InetSocketAddress addr = findLeader();
  connectToLeader(addr);
  // 握手的第一步
  long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
  long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
  if (newEpoch < self.getAcceptedEpoch()) { exception }

  syncWithLeader(newEpochZxid);
  QuorumPacket qp = new QuorumPacket();

  while (this.isRunning()) {
    readPacket(qp);
    processPacket(qp);
  }
}
```

上面方法中，每个子方法的信息量都很大

```java
protected void connectToLeader(InetSocketAddress addr) throws IOException, ConnectException, InterruptedException {
  sock = new Socket();
  sockConnect(sock, addr, Math.min(self.tickTime * self.syncLimit, remainingInitLimitTime));
  sock.setTcpNoDelay(nodelay);
  leaderIs = BinaryInputArchive.getArchive(new BufferedInputStream(sock.getInputStream()));
  bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
  leaderOs = BinaryOutputArchive.getArchive(bufferedOutput);
}

protected long registerWithLeader(int pktType) throws IOException {
  long lastLoggedZxid = self.getLastLoggedZxid();
  QuorumPacket qp = new QuorumPacket();
  qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
  LearnerInfo li = new LearnerInfo(self.getId(), 0x10000, self.getQuorumVerifier().getVersion());
  boa.writeRecord(li, "LearnerInfo");
  qp.setData(bsid.toByteArray());
  writePacket(qp, true);
  readPacket(qp); // expecting learner info

  final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
  if (qp.getType() == Leader.LEADERINFO) {
    // get info
  } else {error}
  // 握手协议，第三部
  QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
  writePacket(ackNewEpoch, true);
  return ZxidUtils.makeZxid(newEpoch, 0);
}

protected void syncWithLeader(long newLeaderZxid) throws Exception {
  QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
  QuorumPacket qp = new QuorumPacket();
  readPacket(qp);
  LinkedList<Long> packetsCommitted = new LinkedList<Long>();
  LinkedList<PacketInFlight> packetsNotCommitted = new LinkedList<PacketInFlight>();
  if (qp.getType() == Leader.DIFF) {

  } else if (qp.getType() == Leader.SNAP) {
    zk.getZKDatabase().deserializeSnapshot(leaderIs);
  } else if (qp.getType() == Leader.TRUNC) {
    boolean truncated = zk.getZKDatabase().truncateLog(qp.getZxid());
  }

  outerLoop:
  while (self.isRunning()) {
    readPacket(qp);
    switch (qp.getType()) {
      case Leader.PROPOSAL:
        PacketInFlight pif = new PacketInFlight();
        packetsNotCommitted.add(pif);
      case Leader.COMMIT:

      case Leader.COMMITANDACTIVATE:
        packetsCommitted.add(qp.getZxid());
      case Leader.INFORM:
      case Leader.INFORMANDACTIVATE:
      case Leader.UPTODATE:
      case Leader.NEWLEADER: // old version zk?
        writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
    }
  }
  writePacket(ack, true);
  zk.startup();
  for (Long zxid : packetsCommitted) {
    fzk.commit(zxid);
  }
}
```
