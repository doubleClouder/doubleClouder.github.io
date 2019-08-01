---
title: weakHashMap与GC
categories: java
date: 2018-03-08 15:56:15
tags: java
---

## 简介
WeakHashMap结构上与HashMap比较类型,内部都是通过Entry[]数组来组织数据，只不过WeakHashMap的Entry[]有些特殊，它的继承体系结构是Entry->WeakReference->Reference，这种结构保证了WeakHashMap的功能。先来看下API文档中对WeakHashMa的描述：以弱键 实现的基于哈希表的 Map。在 WeakHashMap中，当某个键不再正常使用时，将自动移除其条目。更精确地说，对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的丢弃，这就使该键成为可终止的，被终止，然后被回收。丢弃某个键时，其条目从映射中有效地移除。

WeakHashMap的行为取决于垃圾回收器的动作。由于垃圾回收器是由jvm调度的，gc可以发生在WeakHashMap对象生命周期的任何时候，所以WeakHashMap的表现为，即使对 WeakHashMap 实例进行同步，并且没有调用任何赋值方法，在一段时间后 size 方法也可能返回较小的值，对于 isEmpty 方法，可以先返回true，然后返回true，对于给定的键，containsKey方法返回true,然后返回false，对于给定的键,get方法返回一个值,但接着返回null。总而言之就是只要在垃圾回收器清除某个键的弱引用之后，该键才会自动移除。

## 关于引用
深入了解WeakHashMap之前，我们必须对java中的几种引用类型有着明确的认识：

* 强引用

 强引用是相对其他类型引用而言的。如果一个对象具有强引用，GC绝不会回收它，当内存空间不足时，JVM宁愿抛出OOM。new出来的对象是典型的强引用。
 
```
//强引用
Object StrongReference = new Object();
```

* 软引用

 如果一个对象具有软引用，当内存空间不足，GC会回收这些对象的内存，通常可以使用软引用构建敏感数据的缓存。SoftReference中还有个timestamp字段，表示软引用还可以通过设定时间戳进行回收。软引用可以通过get方法获取强引用。声明如下：

```
//SoftReference
SoftReference<Object> sr = new SoftReference<~>(new Object());
Object StrongReference = (Object) SoftRerence.get();
```
* 弱引用
 
 如果一个对象具有弱引用，在GC线程扫描内存区域的过程中，不管当前内存空间足够与否，都会回收内存，使用弱引用 构建非敏感数据的缓存。声明如下：
 
 ```
 //WeakReference
WeakReference<Object> wf = new WeakReference<~>(new Object());
 ```

* 虚引用

 如果一个对象仅持有虚引用，在任何时候都可能被垃圾回收，虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列联合使用，虚引用主要用来跟踪对象被垃圾回收的活动。
 
```
//PhantomReference
PhantomReference<Object> phantomReference=new PhantomReference<Object>(new User(),new ReferenceQueue<Object>());
```

## WeakHashMap与GC

前面我们讲到WeakHashMap中Entry[]有着Entry->WeakReference->Reference这样一个继承结构，我们先来看下Reference类。

```
private static Lock lock = new Lock();
/* List of References waiting to be enqueued.  The collector adds
 * References to this list, while the Reference-handler thread removes
 * them.  This list is protected by the above lock object. The
 * list uses the discovered field to link its elements.
 */
private static Reference<Object> pending = null;
/* High-priority thread to enqueue pending References
 */
private static class ReferenceHandler extends Thread {
    private static void ensureClassInitialized(Class<?> clazz) {
        try {
            Class.forName(clazz.getName(), true, clazz.getClassLoader());
        } catch (ClassNotFoundException e) {
            throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
        }
    }
    static {
        // pre-load and initialize InterruptedException and Cleaner classes
        // so that we don't get into trouble later in the run loop if there's
        // memory shortage while loading/initializing them lazily.
        ensureClassInitialized(InterruptedException.class);
        ensureClassInitialized(Cleaner.class);
    }
    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }
    public void run() {
        while (true) {
            tryHandlePending(true);
        }
    }
}
```
上面是Reference类的部分代码，可以看到Reference中有一个全局的锁对象：lock;有一个静态变量pending;在静态代码块中启动一个ReferenceHandler线程，启动完成后处于wait状态，它在一个Lock同步锁模块中等待。那么WeakHashMap中key/value如何自动回收跟这些有什么关系呢。

我们假设JVM使用cms收集器（使用其他收集器对于弱引用的回收原理相同）。JVM 在进行CMS GC的时候，会创建一个ConcurrentMarkSweepThread（简称CMST）线程去进行GC，ConcurrentMarkSweepThread线程被创建的同时会创建一个SurrogateLockerThread（简称SLT）线程并且启动它，SLT启动之后，处于等待阶段。CMST开始GC时，会发一个消息给SLT让它去获取Java层Reference对象的全局锁：Lock。 直到CMS GC完毕之后，JVM 会将WeakHashMap中所有被回收的对象所属的WeakReference容器对象放入到Reference 的pending属性当中（每次GC完毕之后，pending属性基本上都不会为null了），然后通知SLT释放并且notify全局锁:Lock。此时激活了ReferenceHandler线程的run方法，使其脱离wait状态，开始工作了。ReferenceHandler这个线程会将pending中的所有WeakReference对象都移动到它们各自的列队当中，比如当前这个WeakReference属于某个WeakHashMap对象，那么它就会被放入相应的ReferenceQueue列队里面（该列队是链表结构）。 当我们下次从WeakHashMap对象里面get、put数据或者调用size方法的时候，WeakHashMap就会将ReferenceQueue列队中的WeakReference一一poll出来去和Entry[]数据做比较，如果发现相同的，则说明这个Entry所保存的对象已经被GC掉了，那么将Entry[]内的Entry对象剔除掉，这样就完成的key/value的自动回收。

```
/**
  * Reference queue for cleared WeakEntries
 */
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
```
这是WeakHashMap中的ReferenceQueue定义，注释就可以知道这是用来清除WeakEntries的。

```
/**
    * Expunges stale entries from the table.
    */
   private void expungeStaleEntries() {
       for (Object x; (x = queue.poll()) != null; ) {
           synchronized (queue) {
               @SuppressWarnings("unchecked")
                   Entry<K,V> e = (Entry<K,V>) x;
               int i = indexFor(e.hash, table.length);
               Entry<K,V> prev = table[i];
               Entry<K,V> p = prev;
               while (p != null) {
                   Entry<K,V> next = p.next;
                   if (p == e) {
                       if (prev == e)
                           table[i] = next;
                       else
                           prev.next = next;
                       // Must not null out e.next;
                       // stale entries may be in use by a HashIterator
                       e.value = null; // Help GC
                       size--;
                       break;
                   }
                   prev = p;
                   p = next;
               }
           }
       }
   }
```
expungeStaleEntries方法描述了如何清除，getTable,size等方法会首先调用该方法。以上就是弱引用的清除过程。