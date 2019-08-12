---
title: "Zookeeper in Kafka"
category: Kafka
tag: kafka
---
# Tasks #
Requirements for common-used distributed architecture:
- Master election
It is critical for progress to have a master available to assign tasks to workers.
- Crash detection
The master must be able to detect when workers crash or disconnect.
- Group membership management
The master must be able to figure out which workers are available to execute tasks.
- Metadata management
The master and the workers must be able to store assignments and execution statuses in a reliable manner.

# Basics #
## API Overview ##
ZooKeeper clients connect to a ZooKeeper service and establish a session through which they make API calls. When setting the data of a znode or reading it, the content of the znode is replaced or read entirely.
The ZooKeeper API exposes the following operations:
- create /path data
Creates a znode named with /path and containing data
- delete /path
Deletes the znode /path
- exists /path
Checks whether /path exists
- setData /path data
Sets the data of znode /path to data
- getData /path
Returns the data in /path
- getChildren /path
Returns the list of children under /path
## Different Modes for Znode ##
- Persistent znode
A persistent znode can be deleted only through a call to delete. It's useful when the znode stores some data on behalf of an application and this data needs to be preserved even after its creator is no longer part of the system.
- Ephemeral znode
Ephemeral znode conveys information about some aspect of the application that must exist only while the session of its creator is valid. An ephemeral znode can be deleted when the session of the client creator ends(either by expiration or because it is explicitly closed) or when a client, not necessarily the creator, deletes it.
- Sequential znode
A sequential znode is assigned a unique, monotonically increasing integer. It provides an easy way to create znodes with unique names and also provide a way to easily see the creation order of znodes.
> [How does ZooKeeper's persistent sequential id work](https://bowenli86.github.io/2016/07/07/distributed%20system/zookeeper/How-does-ZooKeeper-s-persistent-sequential-id-work/)

## Watches and Notifications ##
A client can set a watch for changes to the data of a znode, changes to the children of a znode, or a znode being created or deleted.  A watch is a one-time trigger associated with a znode and a type of event (e.g., data is set in the znode, or the znode is deleted). When the watch is triggered by an event, it generates a notification. A notification is a message to the application client that registered the watch to inform this client of the event. 
When an application process registers a watch to receive a notification, the watch is triggered at most once and upon the first event that matches the condition of the watch. One important guarantee of notifications is that they are delivered to a client before any other change is made to the same znode. If a client sets a watch to a znode and there are two consecutive updates to the znode, the client receives the notification after the first update and before it has a chance to observe the second update by, say, reading the znode data. The key property we satisfy is the one that notifications preserve the order of updates the client observes. Although changes to the state of ZooKeeper may end up propagating more slowly to any given client, we guarantee that clients observe changes to the ZooKeeper state according to a global order. 
Each watch is associated with the session in which the client sets it. If the session expires, pending watches are removed. Watches do, however, persist across connections to different servers. Say that a ZooKeeper client disconnects from a ZooKeeper server and connects to a different server in the ensemble. The client will send a list of outstanding watches. When registering the watch, the server will check to see if the watched znode has changed since the watch was registered. If the znode has changed, a watch event will be sent to the client; otherwise, the watch will be registered at the new server.
## Versions ##
Each znode has a version number associated with it that is incremented every time its data changes.

## Sessions ##
The client initially connects to any server in the ensemble, and only to a single server. It uses a TCP connection to communicate with the server, but the session may be moved to a different server if the client has not heard from its current server for some time. Moving a session to a different server is handled transparently by the ZooKeeper client library. All operations a client submits to ZooKeeper are associated to a session.
## Transaction ##
Multiop enables the execution of multiple ZooKeeper operations in a block atomically. The execution is atomic in the sense that either all operations in a multiop block succeed or all fail.
# Use cases #
- Config file subscribe/publish
Store the config data in the znode and subscribe the watcher on it
- Dynamic domain name service
Store the domain-IP map in the znode and subscribe the watcher on it
- Global unique id
Create the sequential znode and get the name as global unique id
- Cluster management
Create the ephemeral znode for each host and subscribe the watcher on it to get notification about it's aliveness. Store host state in the znode.
- Master election
Try to create the ephemeral znode, be master if successfully, be follower if not and subscribe the watcher on it, re-try when get notification about master down
- Distributed lock
Exclusive locks: Try to create the ephemeral znode, got if successfully, subscribe the watcher on it if not, re-try when get notification about lock release
Shared locks: Create sequential ephemeral znode with operation type(read, write), for read operation, subscribe the watcher on last write znode smaller then self, for write operation, subscribe the watcher on last znode smaller than self
- Distributed queue
Create sequential ephemeral znode and subscribe the watcher on last znode smaller than self, do something when become the smallest znode
 
# How Kafka use ZooKeeper #
- Electing a controller. The controller is one of the brokers and is responsible for maintaining the leader/follower relationship for all the partitions. When a node shuts down, it is the controller that tells other replicas to become partition leaders to replace the partition leaders on the node that is going away. Zookeeper is used to elect a controller, make sure there is only one and elect a new one it if it crashes. This is achieved by creating ephemeral znode */controller*
- Cluster membership - which brokers are alive and part of the cluster. This is achieved by creating ephemeral znode under */brokers/ids*. We can also get host, post etc. in it. The clients can register watcher on it to be notified when increase or decrease the brokers.
- Topic configuration - which topic exist, how many partitions each has, where are the replicas, who is the preferred leader, what configuration overrides are set for each topic. This is achieved by creating persistent znode under */brokers/topics* and */config/topics*. The clients can also get the topic metadata from here.
- (0.9.0) - Quotas - how much data is each client allowed to read and write
In order to have different producing and consuming quotas, Kafka Broker allows some clients. This value is set in ZK under /config/clients path. Also, we can change it in bin/kafka-configs.sh script.
- (0.9.0) - ACLs - who is allowed to read and write to which topic.
# Asynchronous operations #
In ZooKeeper, all synchronous calls have corresponding asynchronous calls. 
```java
/**
 * The asynchronous version of create.
 * @see #create(String, byte[], List, CreateMode)
 */
public void create(final String path, byte data[], List<ACL> acl, CreateMode createMode, StringCallback cb, Object ctx)
{
    final String clientPath = path;
    PathUtils.validatePath(clientPath, createMode.isSequential());
    EphemeralType.validateTTL(createMode, -1);

    final String serverPath = prependChroot(clientPath);

    RequestHeader h = new RequestHeader();
    h.setType(createMode.isContainer() ? ZooDefs.OpCode.createContainer : ZooDefs.OpCode.create);
    CreateRequest request = new CreateRequest();
    CreateResponse response = new CreateResponse();
    ReplyHeader r = new ReplyHeader();
    request.setData(data);
    request.setFlags(createMode.toFlag());
    request.setPath(serverPath);
    request.setAcl(acl);
    cnxn.queuePacket(h, r, request, response, cb, clientPath, serverPath, ctx, null);
}
interface StringCallback extends AsyncCallback {
    /**
     * Process the result of the asynchronous call.
     * <p>
     * On success, rc is
     * {@link org.apache.zookeeper.KeeperException.Code#OK}.
     * <p>
     * On failure, rc is set to the corresponding failure code in
     * {@link org.apache.zookeeper.KeeperException}.
     * <ul>
     * <li>
     * {@link org.apache.zookeeper.KeeperException.Code#NODEEXISTS}
     * - The node on give path already exists for some API calls.
     * </li>
     * <li>
     * {@link org.apache.zookeeper.KeeperException.Code#NONODE}
     * - The node on given path doesn't exist for some API calls.
     * </li>
     * <li>
     * {@link
     * org.apache.zookeeper.KeeperException.Code#NOCHILDRENFOREPHEMERALS}
     * - an ephemeral node cannot have children. There is discussion in
     * community. It might be changed in the future.
     * </li>
     * </ul>
     *
     * @param rc   The return code or the result of the call.
     * @param path The path that we passed to asynchronous calls.
     * @param ctx  Whatever context object that we passed to
     *             asynchronous calls.
     * @param name The name of the Znode that was created.
     *             On success, <i>name</i> and <i>path</i> are usually
     *             equal, unless a sequential node has been created.
     */
    public void processResult(int rc, String path, Object ctx, String name);
}
```
The asynchronous method simply queues the request to the LinkedBlockingQueue ***outgoingQueue*** using ***queuePacket*** of ***ClientCnxn***, there is another ***sendThread*** send the packet uses ***doTransport*** of ***ClientCnxnSocketNIO***.
When responses are received, they are queued in the LinkedBlockingQueue ***waitingEvents*** of ***EventThread*** and processed in the order they are received in the single event thread to preserve order.

> [ZooKeeper](https://github.com/Leon-WTF/leon-wtf.github.io/blob/master/doc/zookeeper.pdf)
