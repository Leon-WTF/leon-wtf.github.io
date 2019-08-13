---
title: "ZooKeeper Atomic Broadcast Implementation"
category: ZooKeeper
tag: zookeeper
---
# Fast Leader Election #
Whenever the ***QuorumPeer*** changes its state to LOOKING, ***lookForLeader*** in ***FastLeaderElection*** is invoked to start a new round of leader election. The server with lastest zxid(if same then lasted sid) will be selected.

```java
// Increase the election round number, all valid votes must include the same round number
AtomicLong logicalclock = new AtomicLong();
logicalclock.incrementAndGet();
// Initialize the proposal, then send it
updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
sendNotifications();
// The proposal format as below:
ToSend notmsg = new ToSend(ToSend.mType.notification,
                proposedLeader, // proposed server id, self's server id at first 
                proposedZxid, // proposed server's last zxid, self's last zxid at first
                logicalclock.get(), // self's round number
                QuorumPeer.ServerState.LOOKING, // self's stat
                sid,
                proposedEpoch, // proposed server's round number, read from CURRENT_EPOCH_FILENAME at first
                qv.toString().getBytes());
// Receive the notifications
Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);
// Check the round number
if (n.electionEpoch > logicalclock.get()) {
    logicalclock.set(n.electionEpoch);
    recvset.clear();
    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
        updateProposal(n.leader, n.zxid, n.peerEpoch);
    } else {
        updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
    }
    sendNotifications();
} else if (n.electionEpoch < logicalclock.get()) {
    if(LOG.isDebugEnabled()){
        LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                + Long.toHexString(n.electionEpoch)
                + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
    }
    break;
} else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
        proposedLeader, proposedZxid, proposedEpoch)) {
    updateProposal(n.leader, n.zxid, n.peerEpoch);
    sendNotifications();
}
// Compare the proposal， return true if self's proposal needs to be changed
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    /* We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same as current zxid, but server id is higher.
     */
    return ((newEpoch > curEpoch) ||
            ((newEpoch == curEpoch) &&
            ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
}
// Update proposal and send it if needed
updateProposal(n.leader, n.zxid, n.peerEpoch);
sendNotifications();
// Note down all proposals received and check if a quorum has accepted the proposal, if yes, change the state and stop, if no, go back to receive notifications
recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));
if (voteSet.hasAllQuorums()) {
    // Wait 200ms to see if there is any better proposed leader
    while((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null){
        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)){
            recvqueue.put(n);
            break;
        }
    }
	setPeerState(proposedLeader, voteSet);
	Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
	leaveInstance(endVote);
	return endVote;
}
```

# Synchronize data #

![synchronize-leader](https://img-blog.csdnimg.cn/20190801172304696.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjkwOTA1NQ==,size_16,color_FFFFFF,t_70)
![synchronize-follower](https://img-blog.csdnimg.cn/20190801172347856.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjkwOTA1NQ==,size_16,color_FFFFFF,t_70)

# Broadcast #

![broadcast](https://img-blog.csdnimg.cn/20190801172454666.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjkwOTA1NQ==,size_16,color_FFFFFF,t_70)
> [ZooKeeper’s atomic broadcast protocol: Theory and practice](https://github.com/Leon-WTF/leon-wtf.github.io/blob/master/doc/zookeeper-atomic-broadcast-protocol.pdf)
