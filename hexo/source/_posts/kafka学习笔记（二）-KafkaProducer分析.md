---
title: kafka学习笔记（二）- KafkaProducer分析
cetegories: kafka
date: 2017-11-20 16:22:20
tags: kafka
---
## 前言
自kafka 0.8.2版本以后，kafka团队发布了新的java版本的client api以替代之前的scala版本的api。我们对kafka client端的分析都是建立在java版本的api基础之上。博主分析的kafka版本是0.11.0。

## Producer异步发送

0.8.2之前的scala客户端版本分别实现同步发送和异步发送两种方式。而在0.8.2之后的java版本中同步发送实际上也是通过异步发送间接实现的。

异步发送实际上就是在客户端维护一个消息缓冲区，producer发送消息的时候先将消息保存到消息缓冲区，然后后台发送线程会批量将消息缓冲区的消息发送到broker。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/kafka_producer.png
)
上图我们可以看出kafkaProducer在需要进行消息发送的时候，先把消息放到本地队列，然后由IO线程Sender取出队列的消息，异步的将消息发送给kafka broker。这里的sender其实还干了另外一件事，就是从broker获取meta信息，metaData就是broker,topic和partition之间的映射关系,每一个topic下partion有哪些；partion在broker上是如何分布的；partion的的leader和follwer信息。

从上面的producer的异布发送模型可以看出Sender每次从broker获取meta信息都会写到metaData中，然后kafkaProducer从metaData中获取这些信息，下面我们通过代码来看下metaData的结构。

```
public final class Metadata {
    private static final Logger log = LoggerFactory.getLogger(Metadata.class);
    public static final long TOPIC_EXPIRY_MS = 5 * 60 * 1000;
    private static final long TOPIC_EXPIRY_NEEDS_UPDATE = -1L;
    //更新失败，下一次更新的时间
    private final long refreshBackoffMs;
    //过期时间，也即每隔多久需要更新一次。默认10分钟
    private final long metadataExpireMs;
    //成功更新，版本号递增
    private int version;
    //上一次更新时间
    private long lastRefreshMs;
    //上一次成功更新时间
    private long lastSuccessfulRefreshMs;
    //集群信息
    private Cluster cluster;
    private boolean needUpdate;
    /* Topics with expiry time */
    private final Map<String, Long> topics;
    private final List<Listener> listeners;
    private final ClusterResourceListeners clusterResourceListeners;
    private boolean needMetadataForAllTopics;
    private final boolean allowAutoTopicCreation;
    private final boolean topicExpiryEnabled;
    public Metadata(long refreshBackoffMs, long metadataExpireMs, boolean allowAutoTopicCreation) {
        this(refreshBackoffMs, metadataExpireMs, allowAutoTopicCreation, false, new ClusterResourceListeners());
    }
```
MetaData中包含一些metaData本身操作信息和集群信息Cluster。cluster中真正维护着上面所说的boker与topic/partition的映射关系。

```
public final class Cluster {
    private final boolean isBootstrapConfigured;
    private final List<Node> nodes;
    private final Set<String> unauthorizedTopics;
    private final Set<String> internalTopics;
    private final Node controller;
    private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;
    private final Map<String, List<PartitionInfo>> partitionsByTopic;
    private final Map<String, List<PartitionInfo>> availablePartitionsByTopic;
    private final Map<Integer, List<PartitionInfo>> partitionsByNode;
    private final Map<Integer, Node> nodesById;
    private final ClusterResource clusterResource;
```
在MetaData中，所有的公共方法都是synchronized的，metaData的操作是同步的，所以在多个producer thread读，sender线程写的情况下不会出现线程安全问题。

清楚metaData的结构之后，我们继续看下Sender是什么，Sender是一个IO线程，负责与broker交互，那么它是何时创建的呢。

```
@SuppressWarnings({"unchecked", "deprecation"})
    private KafkaProducer(ProducerConfig config, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
        try {
            log.trace("Starting the Kafka producer");
            
  ····
                       this.metadata.update(Cluster.bootstrap(addresses), Collections.<String>emptySet(), time.milliseconds());
            ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(config);
            Sensor throttleTimeSensor = Sender.throttleTimeSensor(metrics);
            NetworkClient client = new NetworkClient(
                    new Selector(config.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG),
                            this.metrics, time, "producer", channelBuilder),
                    this.metadata,
                    clientId,
                    maxInflightRequests,
                    config.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                    config.getLong(ProducerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),
                    config.getInt(ProducerConfig.SEND_BUFFER_CONFIG),
                    config.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),
                    this.requestTimeoutMs,
                    time,
                    true,
                    apiVersions,
                    throttleTimeSensor);
            this.sender = new Sender(client,
                    this.metadata,
                    this.accumulator,
                    maxInflightRequests == 1,
                    config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG),
                    acks,
                    retries,
                    this.metrics,
                    Time.SYSTEM,
                    this.requestTimeoutMs,
                    config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG),
                    this.transactionManager,
                    apiVersions);
            String ioThreadName = "kafka-producer-network-thread" + (clientId.length() > 0 ? " | " + clientId : "");
            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
            this.ioThread.start();
            this.errors = this.metrics.sensor("errors");
            config.logUnused();
            AppInfoParser.registerAppInfo(JMX_PREFIX, clientId);
            log.debug("Kafka producer started");
        } catch (Throwable t) {
            // call close methods if internal objects are already constructed this is to prevent resource leak. see KAFKA-2121
            close(0, TimeUnit.MILLISECONDS, true);
            // now propagate the exception
            throw new KafkaException("Failed to construct kafka producer", t);
        }
}
```
kafkaProducer的初始化过程是一些序列化规则，压缩类型，监控等基础配置的生成以及sender线程的创建，我们继续看下sender线程具体的动作。

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
            ...
}
```
在sender run方法中我们关注client.poll()这一行，前面transactionManager跟事务性producer相关，这里我们不关注。
client.poll()是调的NetworkClient的poll()方法，这是kafka客户端网络通信的的封装，封装了NIO操作。

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
 }
```
handleCompletedReceives()是对broker response的处理handler

```
private void handleCompletedReceives(List<ClientResponse> responses, long now) {
        for (NetworkReceive receive : this.selector.completedReceives()) {
            String source = receive.source();
            InFlightRequest req = inFlightRequests.completeNext(source);
            Struct responseStruct = parseStructMaybeUpdateThrottleTimeMetrics(receive.payload(), req.header,
                throttleTimeSensor, now);
            if (log.isTraceEnabled()) {
                log.trace("Completed receive from node {}, for key {}, received {}", req.destination,
                    req.header.apiKey(), responseStruct.toString());
            }
            AbstractResponse body = createResponse(responseStruct, req.header);
            if (req.isInternalRequest && body instanceof MetadataResponse)
                metadataUpdater.handleCompletedMetadataResponse(req.header, now, (MetadataResponse) body);
            else if (req.isInternalRequest && body instanceof ApiVersionsResponse)
                handleApiVersionsResponse(responses, req, now, (ApiVersionsResponse) body);
            else
                responses.add(req.completed(body, now));
        }
}
```
这里我们关注metadataResponse的处理

```
@Override
        public void handleCompletedMetadataResponse(RequestHeader requestHeader, long now, MetadataResponse response) {
            this.metadataFetchInProgress = false;
            Cluster cluster = response.cluster();
            // check if any topics metadata failed to get updated
            Map<String, Errors> errors = response.errors();
            if (!errors.isEmpty())
                log.warn("Error while fetching metadata with correlation id {} : {}", requestHeader.correlationId(), errors);
            // don't update the cluster if there are no valid nodes...the topic we want may still be in the process of being
            // created which means we will get errors and no nodes until it exists
            if (cluster.nodes().size() > 0) {
                this.metadata.update(cluster, response.unavailableTopics(), now);
            } else {
                log.trace("Ignoring empty metadata response with correlation id {}.", requestHeader.correlationId());
                this.metadata.failedUpdate(now);
            }
}
```
这里真正执行Metadata的更新

```
public synchronized void update(Cluster cluster, Set<String> unavailableTopics, long now) {
        Objects.requireNonNull(cluster, "cluster should not be null");
        this.needUpdate = false;
        this.lastRefreshMs = now;
        this.lastSuccessfulRefreshMs = now;
        this.version += 1;
        if (topicExpiryEnabled) {
            // Handle expiry of topics from the metadata refresh set.
            for (Iterator<Map.Entry<String, Long>> it = topics.entrySet().iterator(); it.hasNext(); ) {
                Map.Entry<String, Long> entry = it.next();
                long expireMs = entry.getValue();
                if (expireMs == TOPIC_EXPIRY_NEEDS_UPDATE)
                    entry.setValue(now + TOPIC_EXPIRY_MS);
                else if (expireMs <= now) {
                    it.remove();
                    log.debug("Removing unused topic {} from the metadata list, expiryMs {} now {}", entry.getKey(), expireMs, now);
                }
            }
        }
        for (Listener listener: listeners)
            listener.onMetadataUpdate(cluster, unavailableTopics);
        String previousClusterId = cluster.clusterResource().clusterId();
        if (this.needMetadataForAllTopics) {
            // the listener may change the interested topics, which could cause another metadata refresh.
            // If we have already fetched all topics, however, another fetch should be unnecessary.
            this.needUpdate = false;
            this.cluster = getClusterForCurrentTopics(cluster);
        } else {
            this.cluster = cluster;
        }
        // The bootstrap cluster is guaranteed not to have any useful information
        if (!cluster.isBootstrapConfigured()) {
            String clusterId = cluster.clusterResource().clusterId();
            if (clusterId == null ? previousClusterId != null : !clusterId.equals(previousClusterId))
                log.info("Cluster ID: {}", cluster.clusterResource().clusterId());
            clusterResourceListeners.onUpdate(cluster.clusterResource());
        }
        notifyAll();
        log.debug("Updated cluster metadata version {} to {}", this.version, this.cluster);
}
```
上面就是metadata的更新逻辑，实际上kafka实现了两种metadata的更新规则，可以通过producerConfig配置项来完成：

* 周期性的更新: 每隔一段时间更新一次，这个通过 Metadata的lastRefreshMs, lastSuccessfulRefreshMs 这2个字段来实现
配置项为metadata.max.age.ms //缺省300000，即10分钟1次

* 失效检测，强制更新：检查到metadata失效以后，调用metadata.requestUpdate()强制更新。 requestUpdate()函数里面其实什么都没做，就是把needUpdate置成了false。每次poll的时候，都检查这2种更新机制，达到了，就触发更新。

kafkaProducer中调用metadata.requestUpdate()的地方就是主动把metadata主动置为失效。分析以下场景会有此做法：

* 与broker连接初始化的时候
* poll数据的时候，连接断了
* 请求超时
* 消息发送时，有partition的leader没找到
* 返回的response和请求对不上的时候

总之数据不能正常发送或者数据不同步时都认为metadata有问题，需要更新。
还有一点注意的是：更新的时候，是从metadata保存的所有Node，或者说Broker中，选负载最小的那个，也就是当前接收请求最少的那个。向其发送MetadataRequest请求，获取新的Cluster对象。