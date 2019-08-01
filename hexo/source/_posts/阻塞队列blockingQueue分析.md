---
title: 阻塞队列blockingQueue分析
categories: java
date: 2017-07-23 20:22:34
tags: java
---
## 简介
在多线程编程中，生产者消费者是经典的线程间数据共享问题。在多个生产者和消费者的情况，使用队列进行生产和消费的解耦，能有效解决线程间数据共享问题。但是考虑到生产消费速率不均衡的问题，我们必须能够让在队列为空时暂停消费，在队列满时暂停生产；基于这点考虑，我们需要需要在临界值对线程进行阻塞或者唤醒操作，这不仅增加了编程难度，还可能带来带来潜在的线程安全问题。幸好concurrent包为我们提供了强大的BlockingQueue。我们就来看看BlockingQueue如何进行生产消费的。

```
public class BlockingQueueTest {
    public static final BlockingQueue<String> bq = new ArrayBlockingQueue<String>(20);
    public static void main(String[] args)
    {
        for(int i=0; i<10; i++){
            Thread producerThread = new Thread(new Producer());
            producerThread.start();
            Thread consumerThread = new Thread(new Consumer());
            consumerThread.start();
        }
    }
    static class Consumer implements Runnable{
        @Override
        public void run() {
            while (true)
            {
                try
                {
                    System.out.println("线程："+ Thread.currentThread().getName()+"消费数据："+ bq.take());
                    Thread.sleep(1000);
                }
                catch (InterruptedException e)
                {
                    e.printStackTrace();
                }
            }
        }
    }
    static class Producer implements Runnable{
        @Override
        public void run() {
            while (true)
            {
                try
                {
                    System.out.println("线程："+ Thread.currentThread().getName()+"生产数据");
                    bq.put("data");
                    Thread.sleep(1000);
                }
                catch (InterruptedException e)
                {
                    e.printStackTrace();
                }
            }
        }
    }
}
```
我们创建一个容量为10的BlockingQueue，并创建10个生产者和消费者。生产者消费者之间的协调会按照前面所预期的进行。

## BlockingQueue
下面我们就来扒一扒blockingQueue是如何解决我们的难题的。

上面就是BlockingQueue家族的主要成员。blockingQueue作为上层接口，定义了一些核心方法，各成员遵守这些方法的约定进行实现。

* offer(E e) 将对象加入队列中，非阻塞方式。
* offer(E e, long timeout, TimeUnit unit) 设置超时时间，在预期时间内未能放入队列则返回失败。
* put(E e) 阻塞方式入队列，若未能放入则当前线程挂起，直到能够放入线程唤醒。
* add(E e) 如不能放入对象，则抛异常。
* poll() 从队首取出对象，非阻塞方式。
* poll(long timeout, TimeUnit unit)设置取队列元素的等待时间。
* take()阻塞方式取对象。
* remove() 不能取出则抛异常。
* drainTo(Collection<? super E> c) 取出所用可用队列元素。

我们可以发现这里操作是成对的 add(),remove()和offer,poll是对非阻塞队列的操作 而put,take是对阻塞队列的操作。

对blockingQueue中的核心操作有了个概念之后我们以BlockingQueue成员之一ArrayBlockingQueue来看下阻塞队列的实现原理。

```
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    /**
     * Serialization ID. This class relies on default serialization
     * even for the items array, which is default-serialized, even if
     * it is empty. Otherwise it could not be declared final, which is
     * necessary here.
     */
    private static final long serialVersionUID = -817911632652898426L;
    /** The queued items */
    final Object[] items;
    /** items index for next take, poll, peek or remove */
    int takeIndex;
    /** items index for next put, offer, or add */
    int putIndex;
    /** Number of elements in the queue */
    int count;
    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */
    /** Main lock guarding all access */
    final ReentrantLock lock;
    /** Condition for waiting takes */
    private final Condition notEmpty;
    /** Condition for waiting puts */
    private final Condition notFull;
```

ArrayBlockingQueue的队列就是一个数组items。takeIndex和putIndex分别表示当前队首和队尾的下标，count表示队列元素的个数。lock是一个可重入锁，notEmpty和notFull是等待条件。

下面我们直接看下核心操作put()和take()的实现：

```
public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
    
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        notEmpty.signal();
    }
```
首先尝试获取锁，然后判断队列元素个数是否达到容量，没有的话就入队列，否则在条件notFull上等待。入队列enqueue方法中会唤醒在条件notEmpty上等待的线程。

```
public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
    
   private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```
take（）操作则刚好相反，put方法等待的是notFull条件，而take方法等待的是notEmpty条件。条件满足时dequeue，从对首取出元素，唤醒在条件notFull()上等待的线程。

实际上阻塞对列就是用锁代替了我们使用Object.wait()/Object.notify()基于非阻塞队列实现生产消费模型，其他不同类型的BlockingQueue实现原理类似。阻塞队列帮助我们解决了线程同步的难题，避免出错了可能。在认识阻塞队列的作用之后，下一篇我们继续探讨下BlockingQueue家族中另外一位能力出众的成员DelayQueue。
