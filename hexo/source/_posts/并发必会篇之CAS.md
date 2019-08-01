---
title: 并发必会篇之CAS
categories: java
date: 2017-05-02 11:09:30
tags: java
---

## 前言
CAS(compare and swap)在并发领域有着不可或缺的作用，并发大神Doug lea在线程同步中大量使用cas,写出很多颇具艺术性的代码,可以说没有cas，就没有java.util.concurrent包。

CAS的概念想必大家都很清楚，就是字面意思”比较替换”,cas中有三个值：当前内存值V,预期内存值A,要更新的值B。比较内存值V与预期值A是否相同，若是则将V更新成B，否则do nothing。

## 使用
从一个i++问题开始

```
public Test implements Runnable{
    volatile int i;
    public void run() {
        i++;
    }
}
```
我们知道当多个线程并发执行run方法的时候，是无法保证线程安全的。因为i++不是原子操作，即便i有volatile关键字修饰，但是volatile只能保证内存可见性，不能保证线程同步，根据jvm内存模型可以知道最后的执行结果是与预期有偏差的。那么如果得到正确的结果，当然将i++放在一个synchronized方法中来强制线程同步固然可行，但是带来频繁的线程上下文切换的开销，考虑性能并不合适。这里我们使用AtomicInteger来解决。

```
public class Test2 {
    private static AtomicInteger i = new AtomicInteger(0);
    public static void increment() {
        i.getAndIncrement();
    }
}
```
这是我们会发现i的自增在并发情况下总是符合预期的，因为它是原子操作，而这种原子操作正是通过cas来保证。下面通过代码来看下：

```
public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
 //unsafe. getAndAddInt
 public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
        return var5;
    }
```
当两个线程A和B同时执行increment的时候，因为value的原始值是0，也就是在内存中value的值是0，根据jvm内存模型，线程A和B中各自保留一份value副本，值为0。

根据java内存模型,java内存分为工作内存和主存。工作内存即java线程的本地内存，是单独给某个线程分配的，存储局部变量等，同时也会复制主存的共享变量作为本地的副本，目的是为了减少和主存通信的频率，提高效率。线程对共享变量的操作的都是对本地的副本进行操作，在线程执行完毕后刷回主存，而volatile保存内存可见性的含义就是线程每次读取都是从主存读取，更新完成后立即刷回主存，保证线程每次拿共享变量是其他线程的更新对当前线程是可见的。

我们继续分析上面并发执行increment的情况，线程A执行compareAndSwapInt（）从主存拿到value的值是0，而它预期的旧值也是0，所以将value更新成1并刷回主存；线程B同样执行compareAndSwapInt，发现从主存拿到value的值是1，而它预期的值为0，所以本次更新失败，只是将本地value更新成1，继续下一次cas;整个过程中，利用CAS保证了对于value的修改的并发安全。

## 深入原理

compareAndSwapInt是一个本地方法，位于unsafe类中，unsafe类是底层操作为上层java开的一个后门，大多是native方法。

```
public final native boolean compareAndSwapInt(Object paramObject, long paramLong, int paramInt1, int paramInt2);
```
它的实现在unsafe.cpp中，流程如下：先尝试拿到value在内存中的值，然后Atomic::cmpxchg实现即将更新的值和原内存值比较替换。

```
// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}
```
上面是cmpxchg的intel x86的实现。如代码所示（实际是通过别人解释的）程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀，因为单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果。

intel手册对lock前缀的说明如下：

1. 确保后续指令执行的原子性。在Pentium及之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其它处理器暂时无法通过总线访问内存，很显然，这个开销很大。在新的处理器中，Intel使用缓存锁定来保证指令执行的原子性，缓存锁定将大大降低lock前缀指令的执行开销。
2. 禁止该指令与前面和后面的读写指令重排序。
3. 把写缓冲区的所有数据刷新到内存中。

现代cpu的寄存器与内存之间存在L1,L2,L3高速缓存，频繁使用的内存会缓存在高速缓存中，此时以缓存锁定来代替总线锁定，利用缓存一致性机制来保证操作的原子性。从上面二三点我们也可以看出cas同时包含了volatile读和写的内存语义。

## cas存在问题

尽管cas能够实现线程的lock-free,在很多并发场景中提供比锁更优的性能，但是cas也存在如下几个问题：

* ABA问题

如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，则cas操作成功，这样可能会存在问题。

解决办法：在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B-3A。atomic包提供了一个类AtomicStampedReference来解决ABA问题，增加当前标记与预期标志的比较。

* 循环时间长开销大

对于资源竞争严重的情况，CAS自旋的概率会比较大，从而浪费更多的CPU资源。

解决办法：PAUSE指令。PAUSE指令会在循环等待时提示处理器，处理器利用这个提示可以避免在大多数情况下的内存顺序违规，这将大幅提升性能。

* 只能保证一个共享变量的原子操作

解决办法：1.可以用锁。 2.合并这两个变量 Atomic包提供的atomicRefrence应用保证对象之前的原子性，可以把多个变量放在同一个对象里来进行cas操作。