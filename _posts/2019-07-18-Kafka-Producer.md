---
title: "Kafka Producer"
category: Kafka
tag: kafka
---
I will show you the kafka producer workflow using kafka-python:
### Initialize ###

```python
# Create the producer object by passing the config dictionary
producer = KafkaProducer(**kafka_config)
# In the __init__ method of KafkaProducer, it creates below import objects:
# A network client for asynchronous request/response network I/O.
# This is an internal class used to implement the user-facing producer and consumer clients. This class is not thread-safe!
client = KafkaClient(metrics=self._metrics, metric_group_prefix='producer', wakeup_timeout_ms=self.config['max_block_ms'], **self.config)
# This class maintains a dequeue per TopicPartition that accumulates messages into MessageSets to be sent to the server.
# The accumulator attempts to bound memory use, and append calls will block when that memory is exhausted.
self._accumulator = RecordAccumulator(message_version=message_version, metrics=self._metrics, **self.config)
self._metadata = client.cluster
# The background thread that handles the sending of produce requests to the Kafka cluster. This thread makes metadata requests to renew its view of the cluster and then sends produce requests to the appropriate nodes.
self._sender = Sender(client, self._metadata, self._accumulator, self._metrics, guarantee_message_order=guarantee_message_order, **self.config)
# then start sender as a separate daemon thread:
self._sender.daemon = True
self._sender.start()
```
```python
# In __init__ of KafkaClient:
# Check Broker Version if not set explicitly, it will try to connect to bootstrap server and may raise Exception
if self.config['api_version'] is None:
    check_timeout = self.config['api_version_auto_timeout_ms'] / 1000
    self.config['api_version'] = self.check_version(timeout=check_timeout)
# Create a pair connected extra socketa(AF_UNIX) used to wake up selector.select(EpollSelector):
self._wake_r, self._wake_w = socket.socketpair()
self._wake_r.setblocking(False)
self._wake_w.settimeout(self.config['wakeup_timeout_ms'] / 1000.0)
self._selector.register(self._wake_r, selectors.EVENT_READ)
def wakeup(self):
    with self._wake_lock:
        try:
            self._wake_w.sendall(b'x')
        except socket.timeout:
            log.warning('Timeout to send to wakeup socket!')
            raise Errors.KafkaTimeoutError()
        except socket.error:
            log.warning('Unable to send to wakeup socket!')
# In _poll(self, timeout) of KafkaClient:
ready = self._selector.select(timeout)
for key, events in ready:
    if key.fileobj is self._wake_r:
        self._clear_wake_fd()
        continue
    elif not (events & selectors.EVENT_READ):
        continue
    conn = key.data
    processed.add(conn)
def _clear_wake_fd(self):
    # reading from wake socket should only happen in a single thread
    while True:
        try:
            self._wake_r.recv(1024)
        except socket.error:
            break 
```
### Get cluster meta data ###
```python
# When Sender thread is started, its run method is called, then run_once->self._client.poll(poll_timeout_ms)->self._maybe_refresh_metadata()
# If _need_update is not set, Cluster meta data is refreshed every metadata_max_age_ms
ttl = self.cluster.ttl()
def ttl(self):
    """Milliseconds until metadata should be refreshed"""
    now = time.time() * 1000
    if self._need_update:
        ttl = 0
    else:
        metadata_age = now - self._last_successful_refresh_ms
        ttl = self.config['metadata_max_age_ms'] - metadata_age
    retry_age = now - self._last_refresh_ms
    next_retry = self.config['retry_backoff_ms'] - retry_age
    return max(ttl, next_retry, 0)
    
# In the send function of KafkaProducer, _wait_on_metadata is first called to check if partitions exist for the topic, if not, it will set _need_update of ClusterMetadata and wait for max_block_ms, if still can't find, it will raise KafkaTimeoutError:
def _wait_on_metadata(self, topic, max_wait):
    # add topic to metadata topic list if it is not there already.
    self._sender.add_topic(topic)
    begin = time.time()
    elapsed = 0.0
    metadata_event = None
    while True:
        partitions = self._metadata.partitions_for_topic(topic)
        if partitions is not None:
            return partitions

        if not metadata_event:
            metadata_event = threading.Event()

        log.debug("Requesting metadata update for topic %s", topic)

        metadata_event.clear()
        future = self._metadata.request_update()
        future.add_both(lambda e, *args: e.set(), metadata_event)
        self._sender.wakeup()
        metadata_event.wait(max_wait - elapsed)
        elapsed = time.time() - begin
        if not metadata_event.is_set():
            raise Errors.KafkaTimeoutError(
                "Failed to update metadata after %.1f secs." % (max_wait,))
        elif topic in self._metadata.unauthorized_topics:
            raise Errors.TopicAuthorizationFailedError(topic)
        else:
            log.debug("_wait_on_metadata woke after %s secs.", elapsed)
```
### Serialize ###
```python
# In send function of KafkaProducer
key_bytes = self._serialize(self.config['key_serializer'], topic, key)
value_bytes = self._serialize(self.config['value_serializer'], topic, value)
```
### Partition ###
```python
# In send function of KafkaProducer, use partition (int, optional) optionally specify a partition. If not set, the partition will be selected using the configured 'partitioner'.
def _partition(self, topic, partition, key, value,
               serialized_key, serialized_value):
    if partition is not None:
        assert partition >= 0
        assert partition in self._metadata.partitions_for_topic(topic), 'Unrecognized partition'
        return partition

    all_partitions = sorted(self._metadata.partitions_for_topic(topic))
    available = list(self._metadata.available_partitions_for_topic(topic))
    return self.config['partitioner'](serialized_key,
                                      all_partitions,
                                      available)
# Default partitioner
class DefaultPartitioner(object):
    """Default partitioner.
    Hashes key to partition using murmur2 hashing (from java client)
    If key is None, selects partition randomly from available,
    or from all partitions if none are currently available
    """
    @classmethod
    def __call__(cls, key, all_partitions, available):
        """
        Get the partition corresponding to key
        :param key: partitioning key
        :param all_partitions: list of all partitions sorted by partition ID
        :param available: list of available partitions in no particular order
        :return: one of the values from all_partitions or available
        """
        if key is None:
            if available:
                return random.choice(available)
            return random.choice(all_partitions)

        idx = murmur2(key)
        idx &= 0x7fffffff
        idx %= len(all_partitions)
        return all_partitions[idx]
```
### Accumulate records ###
```python
# In send function of KafkaProducer
# First check the message size(include cls.HEADER_STRUCT.size and cls.MAX_RECORD_OVERHEAD) is smaller than max_request_size and buffer_memory
message_size = self._estimate_size_in_bytes(key_bytes, value_bytes, headers)
self._ensure_valid_record_size(message_size)
# Add to RecordAccumulator then return FutureRecordMetadata
result = self._accumulator.append(tp, timestamp_ms, key_bytes, value_bytes, headers, self.config['max_block_ms'], estimated_size=message_size)
future, batch_is_full, new_batch_created = result
```
```python
# In append function of RecordAccumulator
# self._free is SimpleBufferPool containing a deque of io.BytesIO()
buf = self._free.allocate(size, max_time_to_block_ms)
records = MemoryRecordsBuilder(
    self.config['message_version'],
    self.config['compression_attrs'],
    self.config['batch_size']
)
# ProducerBatch->MemoryRecordsBuilder->DefaultRecordBatchBuilder, the buf here is not used, a new bytearray is created in DefaultRecordBatchBuilder!!!
batch = ProducerBatch(tp, records, buf)
future = batch.try_append(timestamp_ms, key, value, headers)
# self._batches is a dict with on deque per TopicPartition
dq = self._batches[tp]
dq.append(batch)
# In __init__ of DefaultRecordBatchBuilder:
self._buffer = bytearray(self.HEADER_STRUCT.size)
# In append of DefaultRecordBatchBuilder, use batch_size to check if buffer is full, if yes, return None and create a new batch:
main_buffer = self._buffer
if (required_size + len_func(main_buffer) > self._batch_size and not first_message):
    return None
```
### Group by broker(node) and send ###
```python
# In the run_once of Sender
# Get a list of nodes whose partitions are ready to be sent, A destination node is ready to send if:
# There is at least one partition that is not backing off its send and those partitions are not muted (to prevent reordering if max_in_flight_requests_per_connection is set to 1) and any of the following are true:
# *The record set is full
# *The record set has sat in the accumulator for at least linger_ms milliseconds
# *The accumulator is out of memory and threads are blocking waiting for data (in this case all partitions are immediately considered ready).
# *The accumulator has been closed
result = self._accumulator.ready(self._metadata)
ready_nodes, next_ready_check_delay, unknown_leaders_exist = result
# Drain all the data for the given nodes and collate them into a list of batches that will fit within the max_request_size on a per-node basis.
batches_by_node = self._accumulator.drain(self._metadata, ready_nodes, self.config['max_request_size'])
# Transfer the record batches into a list of produce requests on a per-node basis.
requests = self._create_produce_requests(batches_by_node)
# Sending Produce Request
(self._client.send(node_id, request, wakeup=False).add_callback(self._handle_produce_response, node_id, time.time(), batches).add_errback(self._failed_produce, batches, node_id))
# Try to read and write to sockets. This method will also attempt to complete node connections, refresh stale metadata, and run previously-scheduled tasks.
self._client.poll(poll_timeout_ms)
```
### [Compression](https://cwiki.apache.org/confluence/display/KAFKA/Compression) ###
```python
# In the drain function of RecordAccumulator, batch.records.close(MemoryRecordsBuilder)->self._builder.build()(DefaultRecordBatchBuilder)->self._maybe_compress()(DefaultRecordBatchBuilder)
batch = dq.popleft()
batch.records.close()
size += batch.records.size_in_bytes()
ready.append(batch)
batch.drained = now
# In DefaultRecordBatchBuilder, the _maybe_compress compress the whole batch:
def _maybe_compress(self):
    if self._compression_type != self.CODEC_NONE:
        self._assert_has_codec(self._compression_type)
        header_size = self.HEADER_STRUCT.size
        data = bytes(self._buffer[header_size:])
        if self._compression_type == self.CODEC_GZIP:
            compressed = gzip_encode(data)
        elif self._compression_type == self.CODEC_SNAPPY:
            compressed = snappy_encode(data)
        elif self._compression_type == self.CODEC_LZ4:
            compressed = lz4_encode(data)
        compressed_size = len(compressed)
        if len(data) <= compressed_size:
            # We did not get any benefit from compression, lets send
            # uncompressed
            return False
        else:
            # Trim bytearray to the required size
            needed_size = header_size + compressed_size
            del self._buffer[needed_size:]
            self._buffer[header_size:needed_size] = compressed
            return True
    return False
```
### Socket ###
The Kafka producer uses the I/O multiplexing model [Linux I/O model 和 JAVA NIO/AIO](https://leon-wtf.github.io/java/2019/06/19/linux-io-model-java-nio-aio/)
#### Create connection ####
```python
# In sender thread, run->run_once->self._client.poll(poll_timeout_ms)->self._maybe_refresh_metadata()->self.maybe_connect(node_id, wakeup=wakeup)->self._connecting.add(node_id)->Queues a node for asynchronous connection during the next poll()
# In poll(self, timeout_ms=None, future=None) of KafkaClient, Attempt to complete pending connections
for node_id in list(self._connecting):
    self._maybe_connect(node_id)
def _maybe_connect(self, node_id):
    """Idempotent non-blocking connection attempt to the given node id."""
    with self._lock:
        conn = self._conns.get(node_id)
        if conn is None:
            broker = self.cluster.broker_metadata(node_id)
            assert broker, 'Broker id %s not in current metadata' % (node_id,)
            log.debug("Initiating connection to node %s at %s:%s",
                      node_id, broker.host, broker.port)
            host, port, afi = get_ip_port_afi(broker.host)
            cb = WeakMethod(self._conn_state_change)
            conn = BrokerConnection(host, broker.port, afi, state_change_callback=cb, node_id=node_id, **self.config)
            self._conns[node_id] = conn
        # Check if existing connection should be recreated because host/port changed
        elif self._should_recycle_connection(conn):
            self._conns.pop(node_id)
            return False
        elif conn.connected():
            return True
        conn.connect()
        return conn.connected()
# In connect(self) of BrokerConnection:
self._sock = socket.socket(self._sock_afi, socket.SOCK_STREAM)
self._sock.setblocking(False)
self.state = ConnectionStates.CONNECTING
self.config['state_change_callback'](self.node_id, self._sock, self)
# In _conn_state_change(self, node_id, sock, conn) of KafkaClient:
if conn.connecting():
    self._selector.register(sock, selectors.EVENT_WRITE)
elif conn.connected():
    self._selector.modify(sock, selectors.EVENT_READ, conn)
# Then in connect(self) of BrokerConnection:
if self.state is ConnectionStates.CONNECTING:
    # in non-blocking mode, use repeated calls to socket.connect_ex to check connection status
    ret = None
    try:
        ret = self._sock.connect_ex(self._sock_addr)
    except socket.error as err:
        ret = err.errno
# In _poll(self, timeout) of KafkaClient, self._selector is EpollSelector in my case:
ready = self._selector.select(timeout)
```
#### Send Request ####
```python
# In the run_once of Sender, self._client.send->conn.send->conn._send will queue the request internally, and the _handle_produce_response callback is added in the future returned. We will need to call send_pending_requests() to trigger network I/O
future = Future()
self.in_flight_requests[correlation_id] = (future, sent_time)
# In _poll(self, timeout) of KafkaClient:
for conn in six.itervalues(self._conns):
    conn.send_pending_requests()
# Then in send_pending_requests(self) of BrokerConnection calls self._send_bytes_blocking(data), the data is sent in blocking mode(self._sock.settimeout) then change back to non-blocking mode(self._sock.settimeout(0.0)):
def _send_bytes_blocking(self, data):
    self._sock.settimeout(self.config['request_timeout_ms'] / 1000)
    total_sent = 0
    try:
        while total_sent < len(data):
            sent_bytes = self._sock.send(data[total_sent:])
            total_sent += sent_bytes
        if total_sent != len(data):
            raise ConnectionError('Buffer overrun during socket send')
        return total_sent
    finally:
        self._sock.settimeout(0.0)
# In _poll(self, timeout) of KafkaClient, if events is selectors.EVENT_READ, calls self._pending_completion.extend(conn.recv()) to add responses in self._pending_completion. conn.recv() calls _recv(self) in BrokerConnection:
while len(recvd) < self.config['sock_chunk_buffer_count']:
    try:
        data = self._sock.recv(self.config['sock_chunk_bytes'])
        # We expect socket.recv to raise an exception if there are no
        # bytes available to read from the socket in non-blocking mode.
        # but if the socket is disconnected, we will get empty data
        # without an exception raised
        if not data:
            log.error('%s: socket disconnected', self)
            self._lock.release()
            self.close(error=Errors.KafkaConnectionError('socket disconnected'))
            return []
        else:
            recvd.append(data)
    except SSLWantReadError:
        break
    except ConnectionError as e:
        if six.PY2 and e.errno == errno.EWOULDBLOCK:
            break
        log.exception('%s: Error receiving network data'
                      ' closing socket', self)
        self._lock.release()
        self.close(error=Errors.KafkaConnectionError(e))
        return []
    except BlockingIOError:
        if six.PY3:
            break
        self._lock.release()
        raise
# The responses of _recv is a list of (correlation_id, response) decoded from received data and compare with self.in_flight_requests.popleft(), in recv, it is translated to list of (response, future), the future is get from in_flight_requests and stored in _send function of BrokerConnection:
for i, (correlation_id, response) in enumerate(responses):
    try:
        with self._lock:
            (future, timestamp) = self.in_flight_requests.pop(correlation_id)
    except KeyError:
        self.close(Errors.KafkaConnectionError('Received unrecognized correlation id'))
        return ()
    latency_ms = (time.time() - timestamp) * 1000
    if self._sensors:
        self._sensors.request_time.record(latency_ms)
    log.debug('%s Response %d (%s ms): %s', self, correlation_id, latency_ms, response)
    responses[i] = (response, future)
# In poll(self, timeout_ms=None, future=None) of KafkaClient calls self._fire_pending_completed_requests() to trigger the future from _pending_completion
# The future will call _handle_produce_response(Sender)->_complete_batch(Sender)->batch.done(base_offset, timestamp_ms, error)(ProducerBatch)->self.produce_future.success((base_offset, timestamp_ms))->_produce_success(FutureRecordMetadata)->self.success(metadata)(Future)->any callback added by the caller
# In __init__ of ProducerBatch:
self.produce_future = FutureProduceResult(tp)
# In try_append of ProducerBatch:
future = FutureRecordMetadata(self.produce_future, metadata.offset, metadata.timestamp, metadata.crc, len(key) if key is not None else -1, len(value) if value is not None else -1, sum(len(h_key.encode("utf-8")) + len(h_val) for h_key, h_val in headers) if headers else -1)
# Then return the FutureRecordMetadata to append of RecordAccumulator, then to send of KafkaProducer, then to outside caller. 
# So the future(FutureRecordMetadata) returned to the caller is associated to a batch(ProducerBatch), and is called by _handle_produce_response callback added in the future(Future) created in BrokerConnection and stored in BrokerConnection.in_flight_requests.
def _fire_pending_completed_requests(self):
    responses = []
    while True:
        try:
            # We rely on deque.popleft remaining threadsafe
            # to allow both the heartbeat thread and the main thread
            # to process responses
            response, future = self._pending_completion.popleft()
        except IndexError:
            break
        future.success(response)
        responses.append(response)
    return responses
```
### Delivery timeout ###
```python
# There is not delivery timeout config for kafka-python client, I think timeout in get function of FutureRecordMetadata has same function:
def get(self, timeout=None):
    if not self.is_done and not self._produce_future.wait(timeout):
        raise Errors.KafkaTimeoutError(
            "Timeout after waiting for %s secs." % (timeout,))
    assert self.is_done
    if self.failed():
        raise self.exception # pylint: disable-msg=raising-bad-type
    return self.value
# In FutureProduceResult:
self._latch = threading.Event()
def wait(self, timeout=None):
    # wait() on python2.6 returns None instead of the flag value
    return self._latch.wait(timeout) or self._latch.is_set()
```
### SSL(Secure Sockets Layer) ###
In a SSL handshake, the broker will send their cetificate to client, the client will check the validity by checking if it is signed by the trusted CA. Then they will communicate one symmetric key using the SSL asymmetric public-private key which is used for the remained of the secure connection.
#### Create a private certificate authority(CA) ####
```bash
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
# Add the CA cetificate to the trust store of client
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file ca-cert
# If you configure the Kafka brokers to require client authentication by setting ssl.client.auth to requested or required, you must also provide a trust store for the Kafka brokers and add the CA cetificate into it
keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file ca-cert
```
#### Create certificate for brokers ####
```bash
# The keystore file stores the certificate and the private key of the certificate
# Ensure that common name (CN) matches exactly with the fully qualified domain name (FQDN) of the server. The client compares the CN with the DNS domain name to ensure that it is indeed connecting to the desired server, not a malicious one.
keytool -keystore kafka.server.keystore.jks -alias localhost -validity {validity} -genkey
```
#### Sign the cetificate ####
```bash
# Export broker certificate from keystore
keytool -keystore kafka.server.keystore.jks -alias localhost -certreq -file cert-file
# Sign it using CA certificate and CA private key and password
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:test1234
# Add CA certificate and signed broker certificate into broker keystore
keytool -keystore kafka.server.keystore.jks -alias CARoot -import -file ca-cert
keytool -keystore kafka.server.keystore.jks -alias localhost -import -file cert-signed
```
>To use SSL in kafka-python, export pem file from the keystore as [Connect to Apache Kafka from Python using SSL](http://maximilianchrist.com/python/databases/2016/08/13/connect-to-apache-kafka-from-python-using-ssl.html) said.

>[Kafka producer overview](https://dzone.com/articles/kafka-producer-overview)
>
>Python clients:
>
>[confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python) : created by Confluent which is based on C client: [librdkafka](https://github.com/edenhill/librdkafka)
>
>[kafka-python](https://github.com/dpkp/kafka-python): created by open source community which is fully written by python
