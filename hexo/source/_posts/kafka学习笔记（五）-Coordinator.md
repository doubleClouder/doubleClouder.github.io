---
title: kafka学习笔记（五）-Coordinator
categories: kafka
date: 2017-12-29 17:57:34
tags: kafka
---

## 前言
对kafka Producer有了一定认识之后，我们继续聊一聊kafka客户端的另一部分Consumer端有关内容，当然仍然基于新的java版本的KafkaConsumer进行分析，本篇内容分析一下kafka如何对同一个消费组里多个consumer进行管理的，消费者之前是如何协调的。

## rebalance机制

我们知道kafka保证同一消费组中的每个consumer能够消费一个或者多个特定的partition数据，一个partition的数据只能被一个consumer消费；因为每个partition里的消息是有序的，这样可以保证partition中的数据被同一个消费者有序消费；同时consumer只需要和自己消费的partition的broker通信就可以，减少开销。

在kafka中消费者的分区分配策略默认有两种：range和RoundRobin。
假设一个consumer group中有三个消费者C0 C1 C2,要消费的topic包含10个partition p0-p19; Range的分配策略是partition个数除以消费者个数，剩下的从头分配 10/3 = 3 则C0分配 0，1，2，3；C1分配4，5，6；C2分配7，8，9；RoundRobin的分配策略则是按Partion的hashCode进行排序，同时对消费者进行排序，然后按hashCode取模进行分配。

consumer rebalance的触发条件：

* Consumer Group的消费者增加或者减少
* Broker的增加或者减少

Rebalance的步骤如下：

* 根据分配策略重新确定每个消费者分配的partition
* 解除Ci对原来分配的Partition的消费权（i从0开始）
* Ci开始消费新的partition

### 基于zk的rebalance

在kafka0.9版本之前,consumer的rebalance是通过在zookeeper上注册watch完成的。每个consumer创建的时候，会在Zookeeper上的路径为/consumers/[consumer group]/ids/[consumer id]下将自己的id注册到消费组下；然后在/consumers/[consumer group]/ids 和/brokers/ids下注册watch；最后强制自己在消费组启动rebalance。

这种做法很容易带来zk的羊群效应，任何Broker或者Consumer的增减都会触发所有的Consumer的Rebalance，造成集群内大量的调整；同时由于每个consumer单独通过zookeeper判断Broker和consumer宕机，由于zk的脑裂特性，同一时刻不同consumer通过zk看到的表现可能是不一样，这就可能会造成很多不正确的rebalance尝试；除此之外，由于consumer彼此独立，每个consumer都不知道其他consumer是否rebalance成功，可能会导致consumer group消费不正确。

### Coordinator
基于zk的rebalance存在不可避免的羊群效应和脑裂问题，如果不用zk来协调，而是将失败探测和Rebalance的逻辑放到一个高可用的中心，那么上述问题就能得以解决；因此kafka0.9.*的版本重新设计了consumer端，诞生了这样一个高可用中心Coordinator，大大减少了zookeeper负载。

对于每一个Consumer Group，Kafka集群为其从broker集群中选择一个broker作为其coordinator。coordinator主要做两件事：

* 维持group的成员组成。这包括加入新的成员，检测成员的存活性，清除不再存活的成员。
* 协调group成员的行为。

Coordinator有如下几种类型：

* GroupCoordinator：broker端的，每个kafka server都有一个实例，管理部分的consumer group和它们的offset。
* WorkerCoordinator：broker端的，管理GroupCoordinator程序，主要管理workers的分配。
* ConsumerCoordinator：consumer端的，和GroupCoordinator通信的媒介。

ConsumerCoordinator是KafkaConsumer的一个成员，只负责与GroupCoordinator通信，所以真正的协调者还是GroupCoordinator。

### 协调过程分析

1. 消费者在启动或者协调节点故障转移时，消费者发送ConsumerMetadataRequest给bootstrap brokers列表中的任意一个brokers。在ConsumerMetadataResponse中，它接收消费者对应的消费组所属的协调节点的位置信息。
2. Consumer连接Coordinator节点，并发送HeartbeatRequest。如果返回的HeartbeatResponse中返回IllegalGeneration错误码，说明协调节点已经在初始化平衡。消费者就会停止抓取数据，提交offsets，发送JoinGroupRequest给协调节点。在JoinGroupResponse，它接收消费者应该拥有的topic-partitions列表以及当前Consumer Group的新的generation编号。这个时候Consumer Group管理已经完成，Consumer就可以开始fetch数据，并为它拥有的partitions提交offsets。
3. 如果HeartbeatResponse没有错误返回，Consumer会从它上次拥有的partitions列表继续抓取数据。

在coordinator中，实际在还在消费组中定义了一个角色leader。所有的成员都向coordinator注册，由coordinator选出leader,由leader来为组中成员分配资源；单个Kafka集群可能会存在着比broker的数量大得多的消费者和消费者组，而消费者的情况可能是不稳定的，可能会频繁变化，每次变化都需要一次协调，如果由broker来负责实际的协调工作，会给broker增加很多负担。所以，从group memeber里选出来一个做为leader，由leader来执行性能开销大的协调任务, 这样把负载分配到client端，可以减轻broker的压力，支持更多数量的消费组。leader 与follower是不直接通信的。consumer group的成员把它们所订阅的topic发送给coordinator。然后coordinator来选择一个leader, 然后由coordinator把这个group的所有成员的订阅情况发给leader，由leader来执行分区的分配，然后把结果发给coordinator。由coordinator来转发分配的结果到每个group的成员。

这里我们看下消费端负责与Coodinator通信的CosumerCoordinator。

```
protected Map<String, ByteBuffer> performAssignment(String leaderId,
                                                        String assignmentStrategy,
                                                        Map<String, ByteBuffer> allSubscriptions) {
        PartitionAssignor assignor = lookupAssignor(assignmentStrategy);
        if (assignor == null)
            throw new IllegalStateException("Coordinator selected invalid assignment protocol: " + assignmentStrategy);
        Set<String> allSubscribedTopics = new HashSet<>();
        Map<String, Subscription> subscriptions = new HashMap<>();
        for (Map.Entry<String, ByteBuffer> subscriptionEntry : allSubscriptions.entrySet()) {
            Subscription subscription = ConsumerProtocol.deserializeSubscription(subscriptionEntry.getValue());
            subscriptions.put(subscriptionEntry.getKey(), subscription);
            allSubscribedTopics.addAll(subscription.topics());
        }
        // the leader will begin watching for changes to any of the topics the group is interested in,
        // which ensures that all metadata changes will eventually be seen
        this.subscriptions.groupSubscribe(allSubscribedTopics);
        metadata.setTopics(this.subscriptions.groupSubscription());
        // update metadata (if needed) and keep track of the metadata used for assignment so that
        // we can check after rebalance completion whether anything has changed
        client.ensureFreshMetadata();
        isLeader = true;
        log.debug("Performing assignment for group {} using strategy {} with subscriptions {}",
                groupId, assignor.name(), subscriptions);
        Map<String, Assignment> assignment = assignor.assign(metadata.fetch(), subscriptions);
        // user-customized assignor may have created some topics that are not in the subscription list
        // and assign their partitions to the members; in this case we would like to update the leader's
        // own metadata with the newly added topics so that it will not trigger a subsequent rebalance
        // when these topics gets updated from metadata refresh.
        //
        // TODO: this is a hack and not something we want to support long-term unless we push regex into the protocol
        //       we may need to modify the PartitionAssingor API to better support this case.
        Set<String> assignedTopics = new HashSet<>();
        for (Assignment assigned : assignment.values()) {
            for (TopicPartition tp : assigned.partitions())
                assignedTopics.add(tp.topic());
        }
        if (!assignedTopics.containsAll(allSubscribedTopics)) {
            Set<String> notAssignedTopics = new HashSet<>(allSubscribedTopics);
            notAssignedTopics.removeAll(assignedTopics);
            log.warn("The following subscribed topics are not assigned to any members in the group {} : {} ", groupId,
                    notAssignedTopics);
        }
        if (!allSubscribedTopics.containsAll(assignedTopics)) {
            Set<String> newlyAddedTopics = new HashSet<>(assignedTopics);
            newlyAddedTopics.removeAll(allSubscribedTopics);
            log.info("The following not-subscribed topics are assigned to group {}, and their metadata will be " +
                    "fetched from the brokers : {}", groupId, newlyAddedTopics);
            allSubscribedTopics.addAll(assignedTopics);
            this.subscriptions.groupSubscribe(allSubscribedTopics);
            metadata.setTopics(this.subscriptions.groupSubscription());
            client.ensureFreshMetadata();
        }
        assignmentSnapshot = metadataSnapshot;
        log.debug("Finished assignment for group {}: {}", groupId, assignment);
        Map<String, ByteBuffer> groupAssignment = new HashMap<>();
        for (Map.Entry<String, Assignment> assignmentEntry : assignment.entrySet()) {
            ByteBuffer buffer = ConsumerProtocol.serializeAssignment(assignmentEntry.getValue());
            groupAssignment.put(assignmentEntry.getKey(), buffer);
        }
        return groupAssignment;
}
```
ConsumerCoordinator中实现了这样一个方法performAssignment，它的入参的是leaderId以及包含所有组成员及partition的meta信息;assignmentStrategy指定了分配策略，这个实际上上是执行的一开始我们介绍的分区分配策略，默认Range或者RoundRobin；PartitionAssignor实现代表具体的策略。leader计算好各member的分区之后，将结果返回给coordinator。

### Coordinator工作过程

1. 在稳定状态下，协调节点通过故障检测协议跟踪每个消费组中每个消费者的健康状况。
2. 在选举和启动时，协调节点读取它管理的消费组列表，以及从ZK中读取每个消费组的成员信息。如果之前没有成员信息，它不会做任何动作。只有在同一个消费组的第一个消费者注册进来时，协调节点才开始工作(即开始加载消费组的消费者成员信息)。
3. 当协调节点完全加载完它所负责的消费组列表的所有组成员之前，它会在以下几种请求的响应中返回CoordinatorStartupNotComplete错误码：HeartbeatRequest，OffsetCommitRequest，JoinGroupRequest。这样消费者就会过段时间重试(直到完全加载，没有错误码返回为止)，知道消费者与coordinator建立连接。
4. 在选举或启动时，协调节点会对消费组中的所有消费者进行故障检测。根据故障检测协议被协调节点标记为Dead的消费者会从消费组中移除，这个时候协调节点会为Dead的消费者所属的消费组触发一个平衡操作(消费者Dead之后，这个消费者拥有的partition需要平衡给其他消费者)。
5. 当HeartbeatResponse返回IllegalGeneration错误码，就会触发平衡操作。一旦所有存活的消费者通过JoinGroupRequests重新注册到协调节点，协调节点会将最新的partition所有权信息在JoinGroupResponse的每个消费者之间通信(同步)，然后就完成了平衡操作。
6. 协调节点会跟踪任何一个消费者已经注册的topics的topic-partition的变更。如果它检测到某个topic新增的partition，就会触发平衡操作。当创建一个新的topics也会触发平衡操作，因为消费者可以在topic被创建之前就注册它感兴趣的topics。

## 总结

afka是使用一个broker作为coordinator来动态协调消费组各消费者，替代之后消费者自己在zookeeper注册watch的方式，提高了高用性，大大降低了zk的压力。

参考： [Kafka源码分析 KafkaConsumer（翻译和注释）](http://zqhxuyuan.github.io/2016/02/22/2016-02-22-Kafka-Consumer-new/)


