---
title: 并发必会篇之ThreadLocal
categories: java
date: 2017-05-13 18:34:02
tags: java
---
## 前言

我们知道在并发编程中，对共享变量的访问必须是非常小心的，因为少有不慎就可能引起线程安全问题，在保证线程安全的前提下，我们往往需要考虑并发场景的性能问题。但在某些场景下，变量是同一个,但是每个线程只需要使用变量的一个副本，独立变化，这时候也就不用考虑并发问题，这就是threadLocal的作用。

## ThreadLocal
threadLocal经典的应用场景是用来解决数据库连接、session管理。
我们先看这样一个数据库连接的例子：

```
class ConnectionManager {
     
    private static Connection connect = null;
     
    public static Connection openConnection() {
        if(connect == null){
            connect = DriverManager.getConnection();
        }
        return connect;
    }
     
    public static void closeConnection() {
        if(connect!=null)
            connect.close();
    }
}
```
这是一个数据连接管理类，很明显在并发情况有线程安全问题：可能出现多个线程同时调用openConnection从而创建多个连接，导致连接泄露；再有一个线程使用连接时，另一个线程调用了close方法，导致连接失效。实际上数据库连接并不需要是单例的，每个线程只要能独立维护自己的连接就可以，但是将OpenConnection方法改成每次调用都创建新的连接，会造成连接频繁创建销毁，服务端压力过重。

这时候使用ThreadLocal可以方便的解决：

```
private static ThreadLocal<Connection> connectionHolder
= new ThreadLocal<Connection>() ;
public Connection getConnection() {
    Connection conn = (Connection)connectionHolder.get();
    if(conn == null){
    conn = connectionManager.getConnection();
    connectionHolder.set(conn)
   }
   return conn;
};
 
public static Connection getConnection() {
return connectionHolder.get();
}
```
这样每个线程都会创建自己的连接,避免了并发问题，同时也避免了连接泛滥；ThreadLocal说白了就是为了线程绑定公共资源，其目的就是为了是每个线程能够隔离公共资源。

下面从代码层面分析一下ThreadLocal如何实现这种隔离的：

```
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
```

这样每个线程都会创建自己的连接,避免了并发问题，同时也避免了连接泛滥；ThreadLocal说白了就是为了线程绑定公共资源，其目的就是为了是每个线程能够隔离公共资源。

下面从代码层面分析一下ThreadLocal如何实现这种隔离的：

```
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
上面是threadLocal的get方法，当我们调用get的时候，首先会获取当前线程，然后拿到线程的ThreadLocalMap。Thread中维护一个threadLocalMap,它是ThreadLocal的一个静态内部类,它以threadLocal对象作为键，值为该threadLocal在线程内部的value;

```
//Thread
ThreadLocalMap threadLocals = null;
 ThreadLocalMap inheritableThreadLocals = null;
```
继续看get方法，如果map不为空并且map中存在该threadLocal的值则直接取出value,否则进行setInitialValue().其实就是在线程内存创建一个ThreadLocalMap的过程。

```
private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

## ThreadLocalMap

每个线程内部都一个一个ThreadLocal.ThreadLocalMap对象，它是ThreadLocal一个内部类，保存线程用到的所有threadLocal及其value。

TheadLocalMap.Entry继承weakReference,是key(ThreadLocal对象)和value线程内部值的容 器。一旦threadLocal对象没有了强引用，所有线程中map中的key就会被回收，value此时不会被回收，下次get/set操作时探测到有key被回收的entry时，会回收value,会将value设成null.这个过程和java.util.weakHashMap类似。

计算hashcode是共享一个AtomicInteger的nexthashcode 从0开始以一个间隔自增，然后根据hashcode &(tablelength-1)定位桶。ThreadLocalMap中发生hash冲突时，不是像HashMap这样用链表来解决冲突，而是是将索引++，放到下一个索引处来解决冲突。

ThreadLocalMap的回收随着线程销毁而回收。一旦线程退出，Thread对象被回收，map中的value也就被回收，但是线程池中的线程永远不会被回收，只会阻塞，如果队列中的某个任务向ThreadLocalput了一个对象，但任务结束后并未清空，那么这个对象在ThreadLocal的下一次put或clear前永远不会被GC；这种情况下，假如线程池有200个线程，那么一个ThreadLocal最多可能造成200个对象的内存泄露。