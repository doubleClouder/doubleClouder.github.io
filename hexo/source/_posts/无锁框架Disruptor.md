---
title: 无锁框架Disruptor
date: 2018-04-01 21:02:40
tags: Disruptor
---

## 前言
LMAX是一种新型零售金融交易平台,它能够以很低的延迟产生大量交易。它能够在一个线程里每秒处理六百万的订单，业务逻辑处理完全运行在内存中，而它的核心就是Disruptor。Disruptor是开源的并发框架，它能够在无锁的情况下实现网络的Queue并发操作。

## Disruptor设计原则

* 尽量保持单一写
* 避免使用锁，无锁化（volatile + CAS）
* 避免伪共享

### 为什么要无锁
在多线程下，往往涉及对资源的共享，比如内存、文件、IO等，为了共享资源访问时的安全，我们需要对共享资源加锁，加锁保证了原子性和内存可见性；（原子性保证资源的争用更新；可见性保证某一线程对内存的修改对其他线程是可见的。）但是锁带来一个不可忽视的问题——性能问题，线程因为竞争不到锁而被挂起，获取到锁线程恢复。这个过程会带来很大的开销，同时cpu上下文的切换，可能会造成缓存和指令的丢失。在多线程情况下，加锁的开销往往比无锁多了好几个数量级。

在某些情况，我们通常会考虑用CAS替代锁从而避免锁带来的巨大开销，相对锁而言，CAS是高效的，因为它没有线程上下文切换。但是如果竟态条件一直不能满足，线程就会不停的自旋，这同样会带来性能开销。

### 避免伪共享
在之前的ArrayBlockingQueue分析中，我们会发现它的三个成员takeIndex,putTndex,count很容易放入一个缓存行，这就会带来伪共享问题，关于伪共享的问题网上的分析已经非常之多，这里不多描述。
![](https://clouder123.oss-cn-beijing.aliyuncs.com/falseSharing.png
)

## 原理简介
先来看下disruptor中的一些核心组件：
![](https://clouder123.oss-cn-beijing.aliyuncs.com/disruptor.png
)

* RingBuffer

环形缓冲区，它是disruptor的核心容器。本质上还是一个数组，通过取余操作，形成一个环形数组；初始化RingBuffer时，所有数组元素均被初始化，后续只会更新数组元素；RingBuffer本身并没有锁，生产者和消费者申请操作位置后直接在数据上操作一个生产者时候，申请位置也不用锁
多个生产者时候，通过CAS控制；

* Sequence

通过顺序递增的序号来编号管理通过其进行交换的数据（事件），对数据(事件)的处理过程总是沿着序号逐个递增处理。一个 Sequence 用于跟踪标识某个特定的事件处理者( RingBuffer/Consumer )的处理进度。虽然一个 AtomicLong 也可以用于标识进度，但定义 Sequence 来负责该问题还有另一个目的，那就是防止不同的 Sequence 之间的CPU缓存伪共享(Flase Sharing)问题。

* Sequence Barrier

用于保持对RingBuffer的已发布Sequence 和Consumer依赖的其它Consumer的 Sequence 的引用。 Sequence Barrier 还定义了决定 Consumer 是否还有可处理的事件的逻辑。

* Wait Strategy 定义 Consumer 如何进行等待下一个事件的策略。
* Event 生产者和消费者之间进行交换的数据被称为事件(Event)。
* EventProcessor 持有特定消费者(Consumer)的 Sequence，并提供用于调用事件处理实现的事件循环(Event Loop)。
* EventHandler：Disruptor 定义的事件处理接口，由用户实现，用于处理事件，是 Consumer 的真正实现。

了解disruptor中核心组件的含义之后，我们可能仍然对disruptor没有一个整体的认识。下面我们分析一下disruptor的生产流程。

## 生产流程

单一生产者的情况下，生产端不需要并发控制。首先申请写入m个元素，若是有m个元素可以写入，则返回最大的序列号（这儿主要判断是否会覆盖未读的元素）若是返回的正确，则生产者开始写入元素。

### 多个生产者
多个生产者情况会遇到，会有多个生产者同时写入同一个位置的问题；disruptor的做法是每个线程获取不同的一段数组空间进行操作。这个通过CAS很容易达到（这跟JVM中创建对象内存分配策略tlab类似）。除此之外，这里仍然有个问题：如何防止读取的时候，读到还未写的元素。Disruptor在多个生产者的情况下，引入了一个与Ring Buffer大小相同的buffer：available Buffer。当某个位置写入成功的时候，便把availble Buffer相应的位置置位，标记为写入成功。读取的时候，会遍历available Buffer，来判断元素是否已经就绪。
读的过程：

* 申请读取到序号n
* 若writer cursor >= n，这时仍然无法确定连续可读的最大下标。从reader cursor开始读取available Buffer，一直查到第一个不可用的元素，然后返回最大连续可读元素的位置；
* 消费者读取元素.
![](https://clouder123.oss-cn-beijing.aliyuncs.com/multWriterReader.png)
如图所示，读线程读到下标为2的元素，三个线程Writer1/Writer2/Writer3正在向RingBuffer相应位置写数据，写线程被分配到的最大元素下标是11。读线程申请读取到下标从3到11的元素，判断writer cursor>=11。然后开始读取availableBuffer，从3开始，往后读取，发现下标为7的元素没有生产成功，于是WaitFor(11)返回6。然后，消费者读取下标从3到6共计4个元素。

写的过程：
![](https://clouder123.oss-cn-beijing.aliyuncs.com/multWriterWrite.png
)

* 申请写入m个元素；
* 若是有m个元素可以写入，则返回最大的序列号。每个生产者被分配到一段独享的空间。
* 生产者写入元素，写入元素的同时设置available Buffer里面相应的位置，以标记自己哪些位置是已经写入成功的。

## 数据结构
RingBuffer

```
abstract class RingBufferPad
{
    protected long p1, p2, p3, p4, p5, p6, p7;
}
 
abstract class RingBufferFields<E> extends RingBufferPad
{
    //填充数组头尾数量
    private static final int BUFFER_PAD;
    //数组的第一个元素位置，去掉填充值
    private static final long REF_ARRAY_BASE;
    //数组元素每个的引用大小的N次方
    private static final int REF_ELEMENT_SHIFT;
    private static final Unsafe UNSAFE = Util.getUnsafe();
    static
    {
        //每个元素引用占用的大小，32位是4，64位是8
        final int scale = UNSAFE.arrayIndexScale(Object[].class);
        if (4 == scale)
        {
            //下面会多次左移，表示乘上每个元素大小
            REF_ELEMENT_SHIFT = 2;
        }
        else if (8 == scale)
        {
            REF_ELEMENT_SHIFT = 3;
        }
        else
        {
            throw new IllegalStateException("Unknown pointer size");
        }
        BUFFER_PAD = 128 / scale;
        //包含PAD大小的数据
        REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);
    }
    //取余公式：m % 2^n = m & ( 2^n - 1 )
    private final long indexMask;
    private final Object[] entries;
    //RingBuffer的大小
    protected final int bufferSize;
    protected final Sequencer sequencer;
 
    RingBufferFields(EventFactory<E> eventFactory, Sequencer sequencer)
    {
        this.sequencer = sequencer;
        this.bufferSize = sequencer.getBufferSize();
        if (bufferSize < 1)
        {
            throw new IllegalArgumentException("bufferSize must not be less than 1");
        }
        if (Integer.bitCount(bufferSize) != 1)
        {
            throw new IllegalArgumentException("bufferSize must be a power of 2");
        }
 
        this.indexMask = bufferSize - 1;
        this.entries = new Object[sequencer.getBufferSize() + 2 * BUFFER_PAD];
        fill(eventFactory);
    }
 
    //初始化时候会将每个位置都填充，之后，只会修改值或者修改引用
    private void fill(EventFactory<E> eventFactory)
    {
        for (int i = 0; i < bufferSize; i++)
        {
            entries[BUFFER_PAD + i] = eventFactory.newInstance();
        }
    }
 
    @SuppressWarnings("unchecked")
    protected final E elementAt(long sequence)
    {
        //每个元素的位置 = 数组基址+数组头+引用偏移
        return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
    }
}
```
Sequence其实就是一个64位的volatile值，内部通过缓存行填充，避免多线程访问sequence的伪共享问题。

```
class LhsPadding
{
    //前填充
    protected long p1, p2, p3, p4, p5, p6, p7;
}
 
class Value extends LhsPadding
{
    //真正用的值，是volatile
    protected volatile long value;
}
 
class RhsPadding extends Value
{
    //后填充
    protected long p9, p10, p11, p12, p13, p14, p15;
}
 
public class Sequence extends RhsPadding
{
    static final long INITIAL_VALUE = -1L;
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;
 
    static
    {
        UNSAFE = Util.getUnsafe();
        try
        {
            VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
        }
        catch (final Exception e)
        {
            throw new RuntimeException(e);
        }
```
SingleProducerSequencer单生产者的核心类，写不存在并发.

```
public long next(int n)
{
    if (n < 1)
    {
        throw new IllegalArgumentException("n must be > 0");
    }
    long nextValue = this.nextValue;
    long nextSequence = nextValue + n;
    long wrapPoint = nextSequence - bufferSize;
    long cachedGatingSequence = this.cachedValue;
     
    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
    {
        cursor.setVolatile(nextValue);
        long minSequence;
        //只要wrapPoint大于最小的gatingSequences，那么不断唤醒消费者去消费，并利用LockSupport让出CPU，直到wrapPoint不大于最小的gatingSequences
        while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))
        {
            waitStrategy.signalAllWhenBlocking();
            LockSupport.parkNanos(1L);
        }
 
        this.cachedValue = minSequence;
    }
    this.nextValue = nextSequence;
    return nextSequence;
}
```
多线程写核心类

```
public long next(int n)
{
    if (n < 1)
    {
        throw new IllegalArgumentException("n must be > 0");
    }
    long current;
    long next;
 
    do
    {
        current = cursor.get();
        next = current + n;
        long wrapPoint = next - bufferSize;
        long cachedGatingSequence = gatingSequenceCache.get();
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)
        {
            long gatingSequence = Util.getMinimumSequence(gatingSequences, current);
 
            if (wrapPoint > gatingSequence)
            {
                waitStrategy.signalAllWhenBlocking();
                LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                continue;
            }
 
            gatingSequenceCache.set(gatingSequence);
        }
        //区别是这里，进行CAS的操作
        else if (cursor.compareAndSet(current, next))
        {
            break;
        }
    }
    while (true);
 
    return next;
}
```
waitStrategy。 Disruptor中有很多需要等待的地方，比如生产者需要等待可用的一个RingBuffer上的槽位，比如消费者1与消费者2之间有依赖关系，都依赖于WaitStrategy。disruptor实现了如下几种等待策略

* BlockingWaitStrategy：利用锁、等待机制的策略
* SleepingWaitStrategy：初始重试200，根据重试的次数，选择盲等或者sleep
* BusySpinWaitStrategy：不停的自旋，耗费CPU性能
* YieldingWaitStrategy：直接调用Thread.yield，延时不可控

SequenceBarrier是消费者与Ringbuffer之间建立消费关系的媒介。

```
public long waitFor(final long sequence)
    throws AlertException, InterruptedException, TimeoutException
{
    checkAlert();
 
    long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
 
    if (availableSequence < sequence)
    {
        return availableSequence;
    }
 
    return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
```

## 总结
disruptor为什么快，因为它最大限度的利用起计算机资源，同时避免了锁的开销，线程预分配内存，批量处理，最大限度提高并行度；而且避免伪共享。

参考：

[高性能队列——Disruptor](https://tech.meituan.com/2016/11/18/disruptor.html)

[Disruptor 极速体验
](http://www.cnblogs.com/haiq/p/4112689.html)
 