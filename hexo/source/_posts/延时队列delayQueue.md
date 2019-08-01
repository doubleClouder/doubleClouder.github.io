---
title: 延时队列delayQueue
categories: java
date: 2017-05-09 20:46:48
tags: java
---

## 前言

在探讨我们标题的内容之前，我们先来看下java要实现一个简单的延迟任务该如何实现（只考虑单机情况）。延迟任务是指由一个事件触发，经过一段时间触发另一个事件；比如用户开通会员一分钟后给用户发短信，比如下单五分钟内未支付取消订单。延迟任务没有固定开始时间，它有别于固定周期执行的定时任务。

那么如何实现这种延时执行的任务呢？最简单的方法就是数据库轮询，所有的订单一般存在db或者缓存中，我们通过一个线程或者使用quartz等定时任务周期的扫描订单，找到超时的订单更改状态，这种实现最为简单。但是如果数据量大并且扫描频率高的话会带来严重的性能问题。下面我们看下延迟队列delayQueue是如何解决这个问题的。

## delayQueue

```
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader = null;
    /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     */
    private final Condition available = lock.newCondition();
```
 delayQueue的内部包含四个元素：可重入锁、一个优先级队列、用于优化阻塞通知的线程leader以及用于阻塞通知的Condition。
 
  priorityQueue是一个非阻塞队列，它是一个最小堆的结构，这里不再详细分析。delayQueue其实就是在每次往优先级队列中添加元素,然后以元素的过期值作为排序因子,以此来达到先过期的元素排在对首的目的。

```
public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```
offer方法以加锁的方式往优先级队列中添加元素，如果元素在队首则设置leader为null 并唤醒等待的线程（至于这么做的目的我们等会分析）。
继续看下take操作：

```
public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```
首先加锁，这没什么好说的。然后拿到优先级队列头部元素，如果为null,说明队列为空，则线程阻塞；否则获取该元素的delay值，若delay值等于0，说明该元素到达延迟时间，是可用状态，则调用poll方法取出元素，直接返回；如果delay时间值不为0，也就是还是不可用状态，则释放对队首元素的引用（避免内存泄露）；接着判断leader是否为空，不为空阻塞当前线程（说明已经有线程早一步在等待该元素）。如果leader元素为空的话,把当前线程赋值给leader元素,然后阻塞delay的时间,即等待队首到达可以出队的时间；循环以上步骤。最后在finally中，判断leader为空并且还有后续节点唤醒其他等待消费的线程。

通过以上分析，想必大家已经认识到leader的作用，它是用来用来减少不必要的等待时间。在多个消费者去take的情况，只能有一个线程去等待元素的delay时间然后取走该元素，这个线程就是最先把自己设为leader的线程，其他线程只能等待下一个元素。

单机下不考虑内存占用情况，使用delayQueue是实现延迟任务的一个有效手段，不仅没有定时扫描的巨大开销，还能保证任务延迟率在一个较低的水平。

## wheelTimer

单机情况下，还有一个解决延迟任务行之有效的手段就是时间轮。

时间轮一种很巧妙的数据结构，如上图，就像一个钟盘，有8个“槽”，可以代表未来的一个时间。如果以秒为单位，中间的指针每隔一秒钟转动到新的“槽”上面。我们加入一个新的延迟任务时, 会根据延迟任务的超时时间与时间轮开始时间算出来它应该在的槽位。如果当前指针指在1上面，我们将入一个5秒后执行的任务，那么这个任务就会放在6这个槽上。那如果需要在20秒之后执行怎么办，由于这个环形结构槽数只到8，所以指针需要多转2圈。位置是在2圈之后的5上面（20 % 8 + 1）。这个圈数需要记录在槽中的数据结构里面。这个数据结构最重要的是两个指针，一个是触发任务的函数指针，另外一个是触发的总第几圈数。时间轮可以用简单的数组或者是环形链表来实现。

netty中已经给出了这种时间轮算法的实现HashedWheelTimer，使用也很简单，只需要通过hashedWheelTimer.newTimeOut(TimerTask task, long delay, TimeUnit unit)方法加入延迟任务。
大家有兴趣可以去看一下io.netty.util.HashedWheelTimer的代码，设计相当巧妙。

## 总结

delayQueue的一个主要应用就是延迟任务的调度，不考虑分布式任务情况下，使用延迟队列可以方便地解决；关于延迟任务，我们还给出了一种有效的方案：时间轮。在高负荷情况下，时间轮会比delayQueue延时率更低，因为delayQueue在高负载情况，线程争用更加激烈，并行消费的能力不高。