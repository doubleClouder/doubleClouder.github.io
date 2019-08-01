---
title: jvm源码分析之内存分配
date: 2017-07-01 20:29:13
categories: java
tags: java
---

## new Object

java程序猿找不到对象怎么办，最简单的办法:new一个。（扎心了，对象还是要自己找。。）。new关键字大家都很熟悉，我们使用new来实例化任何我们需要的对象。Object object = new Object()一行代码对象创建完成，但是大家是否有了解过这行代码背后都做了些什么？其实底层jvm做了一系列非常复杂的工作，现在我们就来扒一扒。

创建一个java对象需要三步：声明引用变量、实例化、初始化对象实例。实例化是真正创建一个java对象，分配内存并返回指向改内存的引用。初始化是指调用构造方法，对类的实例数据赋初值。对于Object object = new Object()可以分成两个部分，”Object obj”这部分的语义作用于Java栈中，会在栈帧中的本地变量表中创建一个引用类型的数据obj,而”new Object()”这部分的语义将会作用于java堆中，形成一块存储Object类型的所有实例数据值的内存区域（当然严格的来讲，对象也是可以在栈上分配内存的，jvm中可以借助对象的逃逸分析来决定一些小对象是否在可以进行栈上分配，这样做的目的是为了降低GC回收频率以及提升GC回收效率，但这毕竟只是一种jvm调优的辅助手段，绝大多数对象实例只能在java堆区分配存储）。下面我们就通过源码看一下对象的内存到底是如何分配的。

```
class AAA{
  public static void main(string[] args){
    Object = new Object()
  }
}
```
利用javap命令反编译上面的代码可以得到：

![](https://clouder123.oss-cn-beijing.aliyuncs.com/image.jpg
)
可以看到java中new关键字对应底层jvm中的new指令，下面我们就来看一下jvm中new指令的实现：
![](https://clouder123.oss-cn-beijing.aliyuncs.com/image2.png
)
首先会尝试去常量池的对应位置找指定类（这里是Object类）的instanceKlass对象（可了解jvm中定义类与对象关系的oop-klass模型）。如果找到说明类已经被加载，否则触发类的初始化过程，这里不再详解，我们直接看对象内存是如何分配的，在allocate_instance方法中。

在讲内存分配过程之前，我们先来了解下内存的几种方式，如果不考虑jvm借助逃逸分析来进行栈上分配，绝大多数对象都是在堆上进行分配的，这点毫无疑问。内存分配的方式如下：

* 指针碰撞：如果内存是规整有序的，分配对象内存时只需要移动指针来划分内存区域。
* 空闲列表：如果内存不是规整的，则记录下空闲内存的地址，申请内存时从列表上获取内存地址。
* 快速分配：上述分配方式在多线程环境下，必须让线程分配内存保持同步（通过先cas,后加锁的方式），这样导致内存分配的效率很低，为了解决线程同步的问题，jvm引入了tlab这种快速分配的方式。

先来简单介绍下tlab(threadLocalAllocationBuffer)。tlab是线程在eden上划分的一块私有区域，通过start、end两个指针卡出一块内存空间，这块空间只能由当前线程在上面进行对象内存分配，由一个top指针在这块卡出的区域移动，类似于指针碰撞。tlab并不是线程私有内存，它是卡出一块空间只让当前线程进行内存分配，这块空间是eden的空间，上面分配的对象还是线程共享的。了解了这样一个概念，我们就继续看下内存的代码，入口是上面提到的allocate_instance.

![](https://clouder123.oss-cn-beijing.aliyuncs.com/new1.png
)
方法中会先判断当前类是否实现了finalize方法，如果有则会创建finalizer对象。我们直接跟内存分配的代码：
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new2.png
)
继续看Common_mem_allocate_init方法最终会调到CollectedHeap::common_mem_allocate_noinit方法

![](https://clouder123.oss-cn-beijing.aliyuncs.com/new3.png
)
这里会看到如果使用tlab，会尝试去tlab上分配，jvm默认开启tlab,jvm中线程run方法会首先去初始化一个tlab。如果不使用tlab，则在直接在堆上分配，这个后面再说，我们先看下tlab上分配。
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new4.png
)
这里会首先尝试在tlab上分配，如果分配失败则进行慢速分配。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/new6.png
)
tlab有个最大可浪费空间的字段，如果当前tlab剩余空间大于要分配的空间，则直接分配，否则会看剩余空间是否大于可浪费空间，若是则直接在eden上分配对象空间，若否则重新丢弃当前tlab，重新在堆上开辟一块tlab,这就是上面提到的慢速分配过程。代码如下：
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new5.png
)
本质上在堆上分配对象空间和分配tlab是一样的，这块内存都是临时的下面看一下堆是如何分配tlab的。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/new7.png
)
首先会确保当前JVM没有进行gc,如果正在gc则不进行分配。首先通过自旋+cas这种活锁的方式进行原子分配，上面代码中的par_allocate会调用jvm的cas指令如下：
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new8.png
)
如果无锁的方式分配失败，则继续执行mem_allocate_work下面的代码,无锁分配失败后则以加锁的方式继续尝试分配

![](https://clouder123.oss-cn-beijing.aliyuncs.com/new9.png
)
如果无锁的方式分配失败，则继续执行mem_allocate_work下面的代码,无锁分配失败后则以加锁的方式继续尝试分配
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new10.png
)
first_only代表是否只在新生代分配，根据上面的代码可以看出大的对象这个值为false，在attempt_allocation方法中，会遍历所有的内存代尝试分配内存，如果first_only为true的话则只在新生代进行分配，如果分配失败直接返回。
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new11.png
)

继续看mem_allocate_work下面的代码：
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new12.png
)
如果上面分配内存仍然失败，则继续执行。

1. gc_locker::is_active_and_needs_gc()为真时，表示当前其它线程已经触发了gc；
2. 如果is_tlab为真，表示当前线程正在为局部分配缓冲区申请内存；
3. 如果!gch->is_maximal_no_gc()为真，表示新生代或老年代可以进行内存扩展，扩展完成后，再次尝试从各代中进行分配expand_heap_and_allocate方法。
4. 如果拓展后分配仍然不成功，则继续往下执行
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new13.png
)
如果当前线程没有位于jni的临界区，将释放Java堆的互斥锁，以使得请求gc的线程可以进行gc操作，等所有本地线程退出临界区和gc完成后，将继续循环尝试分配内存。
如果还是分配不成功，则执行GC操作.
![](https://clouder123.oss-cn-beijing.aliyuncs.com/new14.png
)

## 总结
上述就是jvm中关于对象分配内存的整个流程，jvm会优先以快速分配的方式在tlab上进行，如果分配失败进行堆上分配或者执行tlab的refill过程，期间还有gc的参与，最终是为了给对象在堆上分配一块内存。