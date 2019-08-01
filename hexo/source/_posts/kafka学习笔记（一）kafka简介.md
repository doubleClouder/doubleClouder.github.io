---
title: kafka学习笔记（一）kafka简介
categories: kafka
date: 2017-11-19 15:35:52
tags: kafka
---
## 前言
kafka最初有linkedIn公司研发，2011年开源，12年成为Apache软件基金会顶级开源项目。kafka是一种分布式的MQ系统，越来越来的公司使用kafka作为消息中间件，因为kafka有如下不可比拟的特性：

* 强大的处理能力，kafka是基于磁盘的消息存储，并且以顺序读写访问磁盘，避免了随机读写的性能问题，因此，面对海量消息，kafka也可以高效的存储和查询。
* kafka消息分区的方式提高并发能力，而且支持在线分区，易拓展。
* kafka支持为每个分区创建多个副本，其中只有一个leader副本负责读写，其他副本负责有leader同步，副本会均匀的分布在集群的broker上，容灾能力优秀。
* kafka支持批量读写消息，网络利用率高。

在高并发的系统中，服务拆分是为了突破性能瓶颈的一个有效手段，但是各服务之间数据传输的实时性和可靠性成为新的挑战，这时kafka体现了它的强大之处。

## 原理简介
了解kafka到底如何工作的之前，我们先来了解一些概念。

* 消息: 最基本的数据单元，mafka中的消息由key和value组成。key的作用是根据一定的策略，将消息路由到指定的分区，key可以是null；value部分则是真正的消息体，producer会批量发送消息到kafka。
* topic&partition: topic用于存储消息的逻辑概念，宏观上，kafka就是生产者推消息到topic，消费者poll其中的消息进行消费。每个topic划分成多个分区，kafka是以磁盘存储消息的，partition是消息物理上的分组，它实际上是topic下的一个存储目录，目录下的消息存储方式决定了partition是一个有序的队列。kafka保证同一partition的消息是有序的，但不保证partition之前的顺序。同一topic的不同分区会分配在不同的broker上，partition的命名规则为topic名+分区序号。分区是kafka水平拓展的基础，我们可以通过增加服务器并再其上分配partion的方式来增加kafka的并行处理能力。
* segment partition由多个segment组成，物理上segment是partition目录上的真是存储消息的文件。segment file由两部分组成：index file和data file，segement文件命名规则是partition下的第一个segment从0开始，后续的每个offset是前一个segment的最后一条消息的offerset值。数值为64位long大小，前面用0填充。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/kafka-1.png
)
上图是segment的index文件与data文件的映射关系，当要查找partition中offset为x值的消息时，先通过二分查找定位到x所在segment的index文件，然后在通过index中的稀疏索引定位到消息在long中偏移量。

kafka采用顺序磁盘I/O,所以指向最新的segment追加消息，当超出segment大小限制时，kafka会创建新的segement。正是由于kafka基于磁盘的消息存储，所以无论消费者是否已经拉取消息，kafka都会一直保留这些消息，但是为了节省磁盘空间，kafka必须实现周期性地删除旧的消息，kafka实现了两种保留策略，一是根据消息保留的时间，当消息在kafka中保留超过指定时间，可以被删除，另一种是指定topic中消息的数据量，超过一定量删除旧的消息，kafka会启动一个后台线程，定时检查topic中的消息是否可以删除，当达到条件时，直接删除顺序最前的segment就可以了。

* 副本

kafka中partition可以有多个副本，这是对消息的一种备份策略。每个分区至少有一个副本，当分区中只有一个副本时，就只有leader副本。每个分区的副本集合都会选举出一个副本作为leader,选举策略采用多数同意的方式，所以kafka的副本允许N-1的不存活。

* ISR集合

ISR(In-Sync Replica)集合表示目前可用的分区副本集。ISR集合中的副本必须保证两个条件：1.维持与zk的连接。2.最后一条消息的offset与leader的差值不超过指定阀值。有两个参数可以决定是否将ISR移除ISR集合：replica.lag.max.messages和replica.lag.time.max，前者表明副本offset落后leader offset的最大消息数，后者表示follower向leader发送请求的最大时间间隔。如果再一定的周期内follower不能追赶上leader,可能是由于I/O问题导致follower追加复制消息慢于leader拉取速度，这种属于慢副本。还有一种是卡住副本，由于GC或者副本死亡导致follower停止发送复制请求。

* HW&LEO

HW(high watermark)表示消费者拉取消息的高水位，它标记了一个特殊的offset值，意思就是消费者只能拉取HW之前的消息，HW之后的消息对消费者不可见。当ISR集合中全部的follower都拉取的HW指定消息进行同步后，leader会递增HW的值。

LEO(log end offset)是当前副本的最后一条消息的offset。HW和LEO这两个值都是和ISR集合相关的，引用HW是为了避免消息提交后，follower还未拉取消息，leader跑的过快，此时leader宕机导致的消息丢失问题。但ISR集合有效避免了消息同步复制带来的延时问题，当Followe副本延迟过高时，leader副本被踢出ISR集合，消息已然可以快速提交，当leader宕机时，会选举ISR集合中的follower作为新的leader,这新的leader包含hw之前的所有消息，这样既保证消息不会丢失，也解决了消息同步带来的影响。


