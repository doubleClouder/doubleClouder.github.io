---
title: kafka学习笔记（四）zero-copy使用分析
categories: kafka
date: 2017-12-03 16:53:13
tags: kafka
---

## 前言
kafka作为分布式的MQ系统时一个最重要的特性就是高吞吐量、低延迟。kafka每秒可以处理几十万条消息，它的延迟最低只有几毫秒。每个topic可以分多个partition,kafka使用replication机制备份partition数据。Follower partition可以批量的从Leader partition复制数据，而且Leader充分利用磁盘顺序读以及zero copy机制，这样极大的提高复制性能，内部批量写磁盘，大幅减少了Follower与Leader的消息量差。zero copy是nio重要特性之一，它极大提高数据传输效率，本文我们就来分析一下kafka中如果使用这一特性。

## zero copy机制
说到zero copy，我们先来看下传统io服务端对客户端的传输。对于读操作：jvm虚拟机一定会发送一个read（）操作系统级别的方法，由此会产生一个上下文的切换，从程序所在的用户空间切换至系统的内核空间，内核空间向磁盘空间请求数据，通过DMA直接内存访问的方式将数据读取到内核空间缓冲区，此时用户空间是无法直接使用的，所以下面会将这份缓冲数据原封不动的拷贝到用户空间，至此read操作就结束。期间有两次上下文的切换，和两次数据的拷贝。对于写操作：将文件读取之后需要发送给远端socket客户端。同样调用系统级别的write方法，需要将上述读到的用户空间的数据原封不动的拷贝到内核上的socket缓冲区，然后DMA 引擎将数据从该缓冲区传到协议引擎，这一次拷贝独立地、异步地发生 。我们可以发现在一次网络IO过程有着不必要的上下文切换和数据拷贝。

因此操作系统对于IO的一种优化方式是对于读写操作：用户空间向操作系统发送sendfile()方法，后续的操作只会在内核空间完成。对于所有的读写操作将只会有两次的用户空间和内核空间切换，这种操作称为操作系统意义上的零拷贝。而最佳的实现方式是通过DMA的方式拷贝数据到内核缓冲区，将对应的文件描述符写到socket buffer中，包含了内核的缓冲区的地址和数据长度，并不需要将数据拷贝到socket buffer中，只用存文件描述符。最后协议引擎发送数据的时候，从kernel buffer和socket buffer里面读取数据，对于kernel buffer是真实数据的读取，而对于socket buffer是文件描述符的读取，通过gather操作最终一起发送给客户端。

在nio中通过java.nio.channels.FileChannel. transferTo()方法支持zerocopy技术。

## kafa中如何应用
kafka中partition leader到follower的消息同步和consumer拉取partition中的消息都使用到zero cory。Cousumer从broker获取数据时直接使用了FileChannel.transferTo()，直接在内核态进行的channel到channel的数据传输。

下面通过代码分析一下kafka使用zero cory的流程。首先看下kafka.server.KafkaApis.handle方法，它是kafka业务逻辑处理的入口。

```
def handle(request: RequestChannel.Request) {
       ApiKeys.forId(request.requestId) match {
        case ApiKeys.PRODUCE => handleProducerRequest(request)
        case ApiKeys.FETCH => handleFetchRequest(request)
```
进入handleFetchRequest方法，也即消费者获取消息的逻辑。

```
replicaManager.fetchMessages(  
       fetchRequest.maxWait.toLong, 
      fetchRequest.replicaId, 
      fetchRequest.minBytes,  
      authorizedRequestInfo,  
      sendResponseCallback)
```
ReplicaManager 包含所有主题的所有partition消息。大部分针对Partition的操作都是通过该类来完成的。进入fetchMessages中我们会发现

```
val logReadResults = readFromLocalLog(fetchOnlyFromLeader, fetchOnlyCommitted, fetchInfo)
```
这是真正获取broker数据的地方，内部通过Log对象调用log.read(offset, fetchSize, maxOffsetOpt)，这里的log就是segment log的抽象。进入read发现数据是从这里读到的。

```
val fetchInfo = entry.getValue.read(startOffset, maxOffset, maxLength, maxPosition)
```
fetchInfo对象有一个成员是FileMessageSet，它表示用户在这个Partition这一次消费能够拿到的数据集合。FileMessageSet里面有一个很重要的方法。

```
def writeTo(destChannel: GatheringByteChannel, writePosition: Long, size: Int): Int = {
    ......
    
    val bytesTransferred = (destChannel match {
      case tl: TransportLayer => tl.transferFrom(channel, position, count)
      case dc => channel.transferTo(position, count, dc)
    }).toInt
   
    bytesTransferred
}
```
回到上面readFromLocalLog，获取到数据后会被包装成

```
val fetchPartitionData = logReadResults.mapValues(result =>  FetchResponsePartitionData(result.errorCode, result.hw, result.info.messageSet))
responseCallback(fetchPartitionData)
```
fetchPartitionData包含了上面提到的FileMessageSet，里面有真正进行zero copy的tranferTo()方法。上面代码可以发现回调函数responseCallback的参数是fetchPartitionData。

```
def sendResponseCallback(responsePartitionData: Map[TopicAndPartition, FetchResponsePartitionData]) {
      val mergedResponseStatus = responsePartitionData ++ unauthorizedResponseStatus
...
      def fetchResponseCallback(delayTimeMs: Int) {
        val response = FetchResponse(fetchRequest.correlationId, mergedResponseStatus, fetchRequest.versionId, delayTimeMs)
        requestChannel.sendResponse(new RequestChannel.Response(request, new FetchResponseSend(request.connectionId, response)))
      }
}
```
这里可以发现FetchResponsePartitionData 会被封装成一个FetchResponseSend ,然后由requestChannel发送出去。因为Kafka完全应用是NIO的异步机制，我们继续看下数据如何发送到consumer端。

在SocketServer，也就是负责和所有的消费者打交道，建立连接的中枢里，会不断的进行poll操作。

```
override def run() {
    startupComplete()
    while(isRunning) {
      try {
        // setup any new connections that have been queued up
        configureNewConnections()
        // register any new responses for writing
        processNewResponses()
        try {
          selector.poll(300)
        } catch {
          case...
        }
```
首先会注册新的连接，如果有的话。接着就是处理新的响应了。

```
private def processNewResponses() {
    var curr = requestChannel.receiveResponse(id)
    while(curr != null) {
      try {
        curr.responseAction match {         
          case RequestChannel.SendAction =>
            selector.send(curr.responseSend)
            inflightResponses += (curr.request.connectionId -> curr)
          
        }
      } finally {
        curr = requestChannel.receiveResponse(id)
      }
    }
  }
```
requestChannel.receiveResponse 对应上面FetchResponseSend的requestChannel.sendResponse。这里selector.send会将FetchResponseSend注册一个WRITE事件到selector上。SocketServer 会poll队列，一旦对应的KafkaChannel 写操作ready了，就会调用KafkaChannel的write方法：

```
public Send write() throws IOException {
        if (send != null && send(send)) 
     
    ....
 private boolean send(Send send) throws IOException {
        send.writeTo(transportLayer);
        if (send.completed())
            transportLayer.removeInterestOps(SelectionKey.OP_WRITE);
        return send.completed();
    }
```
send方法的入参send对象就是上面我们注册的FetchResponseSend 对象。接着看下send.writeTo()方法。

```
private val sends = new MultiSend(dest, JavaConversions.seqAsJavaList(fetchResponse.dataGroupedByTopic.toList.map {
    case(topic, data) => new TopicDataSend(dest, TopicData(topic,
                                                     data.map{case(topicAndPartition, message) => (topicAndPartition.partition, message)}))
    }))
override def writeTo(channel: GatheringByteChannel): Long = {
    .....    
     written += sends.writeTo(channel)
    ....
  }
```
可以发现send是一个MultiSend对象，里面包含两个重要成员topicAndPartition.partition: 分区
message:FetchResponsePartitionData。

最后进行writeTo的时候是通过上面提到的FileMessageSet的writeTo，其实最后是通过tl.transferFrom(channel, position, count) 来完成最后的数据发送的。

```
@Override
    public long transferFrom(FileChannel fileChannel, long position, long count) throws IOException {
        return fileChannel.transferTo(position, count, socketChannel);
    }
```

## 总结
上述就是zero copy的介绍以及kafka中是如何对这一机制的合理应用和异步NIO来极大提高消息消费效率的过程。

  