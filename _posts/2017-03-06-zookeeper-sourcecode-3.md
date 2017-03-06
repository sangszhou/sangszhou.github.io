---
layout: post
title:  "Zookeeper code part 3 LeaderElection"
date:   "2016-02-14 17:50:00"
categories: zookeeper
keywords: zookeeper
---

Leader election 在 zk 中是比较独立的一个单元，它有自己的端口，专门用来 leader 选举，有自己的通信实例 (QuorumCnxnManager)，并且这段代码写的也比较清晰

目前这个版本的 leader election 算法中只有一个是可用的，那就是 FastLeaderElection

```java
/**
 * Implementation of leader election using TCP. It uses an object of the class
 * QuorumCnxManager to manage connections. Otherwise, the algorithm is push-based
 * as with the other UDP implementations.
 *
 * There are a few parameters that can be tuned to change its behavior. First,
 * finalizeWait determines the amount of time to wait until deciding upon a leader.
 * This is part of the leader election algorithm.
 */
```

comment 给出来选举的一些信息，比如 finalizewait 是等待时间，在 quorum 收集完选票以后会等待一段时间，如果这段时间内没有更新的 leader 就绪，就直接应用当前的选举结果。

```java
public class FastLeaderElection implements Election {
    /**
     * Connection manager. Fast leader election uses TCP for
     * communication between peers, and QuorumCnxManager manages
     * such connections.
     */    
    QuorumCnxManager manager;

    /**
     * Notifications are messages that let other peers know that
     * a given peer has changed its vote, either because it has
     * joined leader election or because it learned of another
     * peer with higher zxid or same zxid and higher server id
     */
    static public class Notification {
    }
    
    /**
     * Messages that a peer wants to send to other peers.
     * These messages can be both Notifications and Acks
     * of reception of notification.
     */    
    static public class ToSend {
        
    }
    
    LinkedBlockingQueue<ToSend> sendqueue;
    LinkedBlockingQueue<Notification> recvqueue;
    Messenger messenger;
    long proposedLeader;
    long proposedZxid;
    long proposedEpoch;
    
    //Check if a pair (server id, zxid) succeeds our current vote
    protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
        // 评判标准就是比较 zxid, epoch, sid
    }
        
    // 判断是否可以得出结论了
    private boolean termPredicate(HashMap<Long, Vote> votes, Vote vote) {}

    /**
     * Starts a new round of leader election. Whenever our QuorumPeer
     * changes its state to LOOKING, this method is invoked, and it
     * sends notifications to all other peers.
     * JMX
     */
    public Vote lookForLeader() throws InterruptedException {
        // 继承的方法，最重要的两个逻辑之一    
    }
    
    protected class Messenger {
        // 复杂，定义收到消息后的逻辑
        class WorkerReceiver extends ZooKeeperThread  {
            QuorumCnxManager manager;
        }
        // 简单，简单的封装消息，然后靠 manager 发送
        class WorkerSender extends ZooKeeperThread {
            QuorumCnxManager manager;
            
        }
    }
}
```

在 LeaderElection 中，最重要的逻辑方法有两个，第一个是 lookupLeader, 第二个是 Messager 中的 MessageReceiver, 他俩定义消息到来之后的处理逻辑, 先看 lookupLeader

```java
class LeaderElection {
    public Vote lookForLeader() throws InterruptedException {
        HashMap<Long, Vote> recvset = new HashMap<Long, Vote>(); // 投票记录器
        HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>(); // ???
    
        // 每次选举之前，更新 logicalclock, 他与 epoch 的关系是什么？
        synchronized(this){
            logicalclock.incrementAndGet();
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }
        
        // 把自己的成员变量封装成 Notification 发送出去
        sendNotifications();
        
        // 阻塞操作，没选出之前就一直在里面循环
        while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
            // 等待 peer 的消息
            Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);
            if (self.getCurrentAndNextConfigVoters().contains(n.sid)) {
                switch (n.state) {
                    case LOOKING:
                        // If notification > current, replace and send messages out
                        if (n.electionEpoch > logicalclock.get()) { // 自己落后了
                            logicalclock.set(n.electionEpoch);
                        
                            // 清空 recvset, why? 因为这些 vote 都是在一个落后的 logicalclock 下完成的，都是无效的
                            recvset.clear();
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                //  对 n 提出的 leader 表示赞同
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                                } else {
                                    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                                }
                            // 对这个节点表示认同，然后号召大家都投他    
                            sendNotifications();
                        } else if (n.electionEpoch < logicalclock.get()) { // 对方落后了
                            break;
                        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                            // 对其表示认同
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                            sendNotifications();
                        }
                        // 更新选票器
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        if (termPredicate(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch))) {
                            // Verify if there is any change in the proposed leader
                            //  有一个等待时间
                                while((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null){
                                    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)){
                                        recvqueue.put(n);
                                        break;
                                    }
                                }
                        }
                        
                        // 投票完成， 返回结果
                        if (n == null) {
                            self.setPeerState((proposedLeader == self.getId()) ? ServerState.LEADING: learningState());
                            Vote endVote = new Vote(proposedLeader, proposedZxid, proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    case OBSERVING:
                        break;
                    case FOLLOWING:
                    case LEADING:
                        if(n.electionEpoch == logicalclock.get()) {
                            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                            // 如果别人的更大，那么承认他，放弃自己当前的状态，但是此时并不投票？
                            if(termPredicate(recvset, new Vote(n.leader,
                                            n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                                            && checkLeader(outofelection, n.leader, n.electionEpoch)) {

                                self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING: learningState());

                                Vote endVote = new Vote(n.leader, n.zxid, n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }                            
                        }
                        
                        /* Before joining an established ensemble, verify that
                         * a majority are following the same leader.
                         * Only peer epoch is used to check that the votes come
                         * from the same ensemble. This is because there is at
                         * least one corner case in which the ensemble can be
                         * created with inconsistent zxid and election epoch
                         * info. However, given that only one ensemble can be
                         * running at a single point in time and that each 
                         * epoch is used only once, using only the epoch to 
                         * compare the votes is sufficient.
                         * 
                         * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
                         */
                        // 这里由谁来读？没看明白
                        outofelection.put(n.sid, new Vote(n.leader, IGNOREVALUE, IGNOREVALUE, n.peerEpoch, n.state));                        
                        if (termPredicate(outofelection, new Vote(n.leader,
                                IGNOREVALUE, IGNOREVALUE, n.peerEpoch, n.state))
                                && checkLeader(outofelection, n.leader, IGNOREVALUE)) {

                            synchronized(this){
                                logicalclock.set(n.electionEpoch);
                                self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING: learningState());
                            }

                            Vote endVote = new Vote(n.leader, n.zxid, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                        break;                   
            }
        }    
    }
    }
}
```

这段逻辑大体上比较清晰，就是简单的转发请求的过程，如果有一个 leader 比别人新，肯定会获得选举权。唯一产生问题的场景是，所有的 node 除了 sid 之前宣布相同，然后每个收到的投票过程是一个接一个的，比如 1 2 3 4 5, 1 先收到 5 再收。。。 先不想了

另一个逻辑是 WorkerReceiver。workerReceiver 相比于 lookupLeader 来说是一种维护的需要，比如收到来自非 voting group 的请求直接返回，如果已经有 Leader 了直接返回，有 config 到来了该如何处理等等，总之是一个判断逻辑，为了减轻 lookupLeader 的工作量

```java
class FastLeaderElection {
    public void run() {
        while (!stop) {
            // manager 的 queue 是底层的东西，更上层的还有一个 queue
            response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
            Notification n = new Notification();
            
            //If it is from a non-voting server (such as an observer or a non-voting follower), respond right away.
            if(!self.getCurrentAndNextConfigVoters().contains(response.sid)) {
                sendqueue.offer(notmsg);
            } else {
                QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                
                // 如果自己的状态 
                if(self.getPeerState() == QuorumPeer.ServerState.LOOKING) {
                    recvqueue.offer(n); // 上传到 queue 里
                    //* Send a notification back if the peer that sent this
                    //* message is also looking and its logical clock is lagging behind.                    
                    if((ackstate == QuorumPeer.ServerState.LOOKING) && (n.electionEpoch < logicalclock.get())) {
                        Vote v = getVote();
                        QuorumVerifier qv = self.getQuorumVerifier();
                        sendqueue.offer(notmsg);
                    } else {
                        //* If this server is not looking, but the one that sent the ack
                        //* is looking, then send back what it believes to be the leader.
                        if(ackstate == QuorumPeer.ServerState.LOOKING){
                          sendqueue.offer(notmsg);  
                        }
                    }
                }
            }
            
        }
    }
}
```

整体来看，LeaderElection 还是比较简单的