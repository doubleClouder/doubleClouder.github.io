---
title: kafka学习笔记（三）-kafkaProducer NIO模型
categories: kafka
date: 2017-11-30 15:58:46
tags: kafka
---
## 前言
在上篇的kafkaProducer分析中，我们知道kafka中的nio模型并没有使性能和易用性上都非常优秀的netty框架，而是自己封装了nio的调用。kafka client端的IO操作是通过NetworkClient来完成，NetworkClient底层封装了java NIO。

## producer网络模型
上一篇内容中也提到，kafkaProducer线程会把消息放入本地队列，真正的IO操作是通过sender线程来完成的，sender作为最上层的接口，调用的是NetworkClient,NetworkClient则触发Selector。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/kafka_nio.png
)
上面展示了kafkaProducer网络模型的层次划分，从上往下依次为：最上层Sender调用层作为整个体系的入口，下面是Network接口层、Network层以及nio层。Sender调用NetworkClient，NetworkClient调用 Selector，而Selector底层封装了Java NIO的相关接口。

## Network层
NetworkClient作为网络层接口，真正的NIO封装是在NetWork层实现。下面我们看一下网络层的组件构成。

* kafkaChannel: 对nio的socketChannel的封装，只是中间多了一层代理TransportLayer，可以实现加密的channel.
* NetworkSend/NetworkReceive是对ByteBuffer的封装，表示一次请求的数据包。
* kafka selector：Kafka的Selector封装了NIO的Selector，内含一个NIO Selector对象。

### Kafka Selector
org.apache.kafka.common.network.Selector构成如下:

```
public class Selector implements Selectable, AutoCloseable {
    public static final long NO_IDLE_TIMEOUT_MS = -1;
    private static final Logger log = LoggerFactory.getLogger(Selector.class);
    private final java.nio.channels.Selector nioSelector;
    private final Map<String, KafkaChannel> channels;
    private final List<Send> completedSends;
    private final List<NetworkReceive> completedReceives;
    private final Map<KafkaChannel, Deque<NetworkReceive>> stagedReceives;
    private final Set<SelectionKey> immediatelyConnectedKeys;
    private final Map<String, KafkaChannel> closingChannels;
    private final Map<String, ChannelState> disconnected;
    private final List<String> connected;
    private final List<String> failedSends;
    private final Time time;
    private final SelectorMetrics sensors;
    private final ChannelBuilder channelBuilder;
    private final int maxReceiveSize;
    private final boolean recordTimePerConnection;
    private final IdleExpiryManager idleExpiryManager;
```
可以看出kafka selector中还包括一个kafkaChannel的连接池。selector中包含几个重要的操作 send和poll。先来看下send.

```
/**
     * Queue the given request for sending in the subsequent {@link #poll(long)} calls
     * @param send The request to send
     */
    public void send(Send send) {
        String connectionId = send.destination();
        if (closingChannels.containsKey(connectionId))
            this.failedSends.add(connectionId);
        else {
            KafkaChannel channel = channelOrFail(connectionId, false);
            try {
                channel.setSend(send);
            } catch (CancelledKeyException e) {
                this.failedSends.add(connectionId);
                close(channel, false);
            }
        }
    }
```
根据数据包的目的地找到对应的channel,然后我们发现数据包并有通过channel发出来，而是通过channel.setSend()暂存在channel中。

```
//kafkaChannel
public void setSend(Send send) {
        if (this.send != null)
        //这里如果channel的当前数据包不为空也就数据包还没有发出去，不能存放下一个，直接抛出异常
            throw new IllegalStateException("Attempt to begin a send operation with prior send operation still in progress.");
        this.send = send;
        this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
}
```
可以看出channel一次只能暂存一个数据包。暂存在channel之后，并注册一个写事件，poll就会进行处理。

```
@Override
    public void poll(long timeout) throws IOException {
        if (timeout < 0)
            throw new IllegalArgumentException("timeout should be >= 0");
        clear();
        if (hasStagedReceives() || !immediatelyConnectedKeys.isEmpty())
            timeout = 0;
        /* check ready keys */
        long startSelect = time.nanoseconds();
        int readyKeys = select(timeout);
        long endSelect = time.nanoseconds();
        this.sensors.selectTime.record(endSelect - startSelect, time.milliseconds());
        if (readyKeys > 0 || !immediatelyConnectedKeys.isEmpty()) {
            pollSelectionKeys(this.nioSelector.selectedKeys(), false, endSelect);
            pollSelectionKeys(immediatelyConnectedKeys, true, endSelect);
        }
        addToCompletedReceives();
        long endIo = time.nanoseconds();
        this.sensors.ioTime.record(endIo - endSelect, time.milliseconds());
        // we use the time at the end of select to ensure that we don't close any connections that
        // have just been processed in pollSelectionKeys
        maybeCloseOldestConnection(endSelect);
    }
    
   ...
 private void pollSelectionKeys(Iterable<SelectionKey> selectionKeys,
                                   boolean isImmediatelyConnected,
                                   long currentTimeNanos) {
        Iterator<SelectionKey> iterator = selectionKeys.iterator();
        while (iterator.hasNext()) {
            SelectionKey key = iterator.next();
            iterator.remove();
            KafkaChannel channel = channel(key);
            long channelStartTimeNanos = recordTimePerConnection ? time.nanoseconds() : 0;
            // register all per-connection metrics at once
            sensors.maybeRegisterConnectionMetrics(channel.id());
            if (idleExpiryManager != null)
                idleExpiryManager.update(channel.id(), currentTimeNanos);
            try {
                /* complete any connections that have finished their handshake (either normally or immediately) */
                if (isImmediatelyConnected || key.isConnectable()) {
                    if (channel.finishConnect()) {
                        this.connected.add(channel.id());
                        this.sensors.connectionCreated.record();
                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        log.debug("Created socket with SO_RCVBUF = {}, SO_SNDBUF = {}, SO_TIMEOUT = {} to node {}",
                                socketChannel.socket().getReceiveBufferSize(),
                                socketChannel.socket().getSendBufferSize(),
                                socketChannel.socket().getSoTimeout(),
                                channel.id());
                    } else
                        continue;
                }
                /* if channel is not ready finish prepare */
                if (channel.isConnected() && !channel.ready())
                    channel.prepare();
                /* if channel is ready read from any connections that have readable data */
                if (channel.ready() && key.isReadable() && !hasStagedReceive(channel)) {
                    NetworkReceive networkReceive;
                    while ((networkReceive = channel.read()) != null)
                        addToStagedReceives(channel, networkReceive);
                }
                /* if channel is ready write to any sockets that have space in their buffer and for which we have data */
                if (channel.ready() && key.isWritable()) {
                    Send send = channel.write();
                    if (send != null) {
                        this.completedSends.add(send);
                        this.sensors.recordBytesSent(channel.id(), send.size());
                    }
                }
                /* cancel any defunct sockets */
                if (!key.isValid())
                    close(channel, true);
            } catch (Exception e) {
                String desc = channel.socketDescription();
                if (e instanceof IOException)
                    log.debug("Connection with {} disconnected", desc, e);
                else
                    log.warn("Unexpected error from {}; closing connection", desc, e);
                close(channel, true);
            } finally {
                maybeRecordTimePerConnection(channel, channelStartTimeNanos);
            }
        }
}
```
pollSelectionKeys方法实现了我们熟悉的nio操作，通过Send send = channel.write()取出channel暂存的数据包并发送出来，然后将完成发送的结果加入输出结果集合。这里channel.write()仍然会返回一个send对象，由于write是非阻塞的，它并不会等整个数据包发送出去才返回，所以当channel.write完成整个数据包发送时会返回发送的send对象，如果没有完全发送出去则返回null。这里不能完全发送的情况时在NetworkClient中，往下传的是一个clientRequest,进入到seletor中，暂存在channel里面的也是一个完整的Send对象，但这个send对象交由底层write的时候并没有一次发送出去，需要多次调用write。

同样的，在接受的时候channel.read() ，一个ClientResponse也需要多次read,所以需要在while中read,并且将消息分包：

```
if (channel.ready() && key.isReadable() && !hasStagedReceive(channel)) {
                    NetworkReceive networkReceive;
                    while ((networkReceive = channel.read()) != null)
                        addToStagedReceives(channel, networkReceive);
                }
         ...
 /**
     * adds a receive to staged receives
     */
    private void addToStagedReceives(KafkaChannel channel, NetworkReceive receive) {
        if (!stagedReceives.containsKey(channel))
            stagedReceives.put(channel, new ArrayDeque<NetworkReceive>());
        Deque<NetworkReceive> deque = stagedReceives.get(channel);
        deque.add(receive);
}
```
selector通过一个queue存储每个数据分包直到一个response完全接收。我们再看看上层NetworkClient是如何处理的：

```
@Override
    public List<ClientResponse> poll(long timeout, long now) {
        if (!abortedSends.isEmpty()) {
            // If there are aborted sends because of unsupported version exceptions or disconnects,
            // handle them immediately without waiting for Selector#poll.
            List<ClientResponse> responses = new ArrayList<>();
            handleAbortedSends(responses);
            completeResponses(responses);
            return responses;
        }
        long metadataTimeout = metadataUpdater.maybeUpdate(now);
        try {
            this.selector.poll(Utils.min(timeout, metadataTimeout, requestTimeoutMs));
        } catch (IOException e) {
            log.error("Unexpected error during I/O", e);
        }
        // process completed actions
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
        handleCompletedSends(responses, updatedNow);
        //更新metaData
        handleCompletedReceives(responses, updatedNow);
        handleDisconnections(responses, updatedNow);
        handleConnections();
        handleInitiateApiVersionRequests(updatedNow);
        handleTimedOutRequests(responses, updatedNow);
        completeResponses(responses);
        return responses;
}
```
上面提到selector的poll会把处理结果放入结果集，这里就是对这些结果集进行处理。在往上层看，从调用层sender的run方法：

```
void run(long now) {
        if (transactionManager != null) {
            if (!transactionManager.isTransactional()) {
                // this is an idempotent producer, so make sure we have a producer id
                maybeWaitForProducerId();
            } else if (transactionManager.hasInFlightRequest() || maybeSendTransactionalRequest(now)) {
                // as long as there are outstanding transactional requests, we simply wait for them to return
                client.poll(retryBackoffMs, now);
                return;
            }
            // do not continue sending if the transaction manager is in a failed state or if there
            // is no producer id (for the idempotent case).
            if (transactionManager.hasFatalError() || !transactionManager.hasProducerId()) {
                RuntimeException lastError = transactionManager.lastError();
                if (lastError != null)
                    maybeAbortBatches(lastError);
                client.poll(retryBackoffMs, now);
                return;
            } else if (transactionManager.hasAbortableError()) {
                accumulator.abortUndrainedBatches(transactionManager.lastError());
            }
        }
        long pollTimeout = sendProducerData(now);
        client.poll(pollTimeout, now);
    }
  
  ...
  private long sendProducerData(long now) {
        Cluster cluster = metadata.fetch();
        // get the list of partitions with data ready to send
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);
        // if there are any partitions whose leaders are not known yet, force metadata update
        if (!result.unknownLeaderTopics.isEmpty()) {
            // The set of topics with unknown leader contains topics with leader election pending as well as
            // topics which may have expired. Add the topic again to metadata to ensure it is included
            // and request metadata update, since there are messages to send to the topic.
            for (String topic : result.unknownLeaderTopics)
                this.metadata.add(topic);
            this.metadata.requestUpdate();
        }
        // remove any nodes we aren't ready to send to
        Iterator<Node> iter = result.readyNodes.iterator();
        long notReadyTimeout = Long.MAX_VALUE;
        while (iter.hasNext()) {
            Node node = iter.next();
            if (!this.client.ready(node, now)) {
                iter.remove();
                notReadyTimeout = Math.min(notReadyTimeout, this.client.connectionDelay(node, now));
            }
        }
        // create produce requests
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes,
                this.maxRequestSize, now);
        if (guaranteeMessageOrder) {
            // Mute all the partitions drained
            for (List<ProducerBatch> batchList : batches.values()) {
                for (ProducerBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }
        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(this.requestTimeout, now);
        boolean needsTransactionStateReset = false;
        // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
        // for expired batches. see the documentation of @TransactionState.resetProducerId to understand why
        // we need to reset the producer id here.
        if (!expiredBatches.isEmpty())
            log.trace("Expired {} batches in accumulator", expiredBatches.size());
        for (ProducerBatch expiredBatch : expiredBatches) {
            failBatch(expiredBatch, -1, NO_TIMESTAMP, expiredBatch.timeoutException());
            if (transactionManager != null && expiredBatch.inRetry()) {
                needsTransactionStateReset = true;
            }
            this.sensors.recordErrors(expiredBatch.topicPartition.topic(), expiredBatch.recordCount);
        }
        if (needsTransactionStateReset) {
            transactionManager.resetProducerId();
            return 0;
        }
        sensors.updateProduceRequestMetrics(batches);
        // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
        // loop and try sending more data. Otherwise, the timeout is determined by nodes that have partitions with data
        // that isn't yet sendable (e.g. lingering, backing off). Note that this specifically does not include nodes
        // with sendable data that aren't ready to send since they would cause busy looping.
        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
        if (!result.readyNodes.isEmpty()) {
            log.trace("Nodes with data ready to send: {}", result.readyNodes);
            // if some partitions are already ready to be sent, the select time would be 0;
            // otherwise if some partition already has some data accumulated but not ready yet,
            // the select time will be the time difference between now and its linger expiry time;
            // otherwise the select time will be the time difference between now and the metadata expiry time;
            pollTimeout = 0;
        }
        sendProduceRequests(batches, now);
        return pollTimeout;
}
```
真正执行消息发送的逻辑在sendProducerData中，首先获取metaData中已经准备好的服务端节点client,通过client.ready(node, now)判断，这里节点准备好需要下面几个条件：

1. metadata正常，不需要update
2. 连接正常 connectionStates.isConnected(node)
3. channel是ready状态：这个对于PlaintextChannel， 一直返回true。
4. 当前该channel中，没有in flight request，所有请求都处理完了
5. 当前该channel中，队列尾部的request已经完全发送出去, request.completed()，并且inflight request数目，没有超过设定的最大值，缺省为5.

这里还有一个问题，在response还没有回来的request，kafka是如何保存，request与response的对应关系的。答案在NetworkClient中，答案就是上面条件中的inFlightRequests。

```
//NetworkClient
/* the set of requests currently being sent or awaiting a response */
    private final InFlightRequests inFlightRequests;
    
final class InFlightRequests {
    private final int maxInFlightRequestsPerConnection;
    private final Map<String, Deque<NetworkClient.InFlightRequest>> requests = new HashMap<>();
```
在InFlightRequests中，存放了所有发出去，但是response还没有回来的request。request发出去的时候，入对列；response回来，就把相对应的request出队列。requests的key是消息的目的地。所以这就要求服务端必须保证消息的有序。即broker收到request的顺序是0，1，2。那么response必须是0，1，2。为此服务端实现了一种类似锁的机制mute/unmute，每当一个channel上面接收到一个request，这个channel就会被mute，然后等response返回之后，才会再unmute。这样就保证了同1个连接上面，同时只会有1个请求被处理。InFlightRequests还有一个作用就是判断连接是否失效。

### 长连接维护
我们知道在所有的tcp长连接里，如果去判断连接是否失效，通过的做法是通过一个单独的线程去维护心跳，但是在kafka client里没有这样的线程，同样底层nio也无法告知你连接是否断开。kafka client里有三种方式来监测：

* 所有的IO函数，connect, finishConnect, read, write都会抛IOException，因此任何时候，调用这些函数的时候，只要抛异常，就认为连接已经断开。
* selectionKey.isValid()
* 上面提到的inflightRequests。所有发出去的request，都设置有一个response返回的时间。在这个时间内，response没有回来，就认为连接断了。

在初始化建立连接的时候和每次selector poll的时候都会去检查连接是否断开。

```
private final Map<String, ChannelState> disconnected;
    private final List<String> connected;
```
上面是kafka selector中保存连接状态的数据结构。 总结：

1. Selector中的连接状态，在每次poll之前，会调用clear清空；在poll之后，收集。
2. Selector中的连接状态，会传给上层NetworkClient，用于它更新自己的连接状态。
3. 除了来自Selctor，NetworkClient自己内部的inflightRequests，也就是上面的手段3， 也用于检测连接状态。

连接失效重连的操作在上层Sender中完成，上面判断一个节点是否ready中会发起失效连接的重连。

```
public boolean ready(Node node, long now) {
        if (node.isEmpty())
            throw new IllegalArgumentException("Cannot connect to empty node " + node);
        if (isReady(node, now))
            return true;
        if (connectionStates.canConnect(node.idString(), now))
            // if we are interested in sending to a node and we don't have a connection to it, initiate one
            initiateConnect(node, now);
        return false;
}
```
这里需要注意的是poll检查到连接断了返回给上层sender，sender只是发起连接，不会等到连接建立好了，再执行下面的代码。而已在poll之后，判断连接是否建立；在下1次或者下几次poll之前，可能连接才会建立好，ready才会返回true.

    

