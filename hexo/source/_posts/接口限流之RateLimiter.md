---
title: 接口限流之RateLimiter
categories: java
date: 2017-07-14 21:03:18
tags: java
---

## 描述

对于高并发的系统而言，缓存、限流和降级是保证服务可用的三大利器。由于缓存的性能越来越高，对于系统能抗住高并发流量的作用自然不必多说；熔断和降级是保证服务可用的最后一道屏障，它是在下游服务故障或者系统负荷过高时不得不做的处理，通过一些mock的处理让服务暂时看起来可用，待高峰过后重新恢复。但在有些场景下，可以用来控制服务的请求速率，如双十一和12306的限流。

## 限流算法

常见的限流算法有漏桶算法和令牌桶算法。

![](http://clouder123.oss-cn-beijing.aliyuncs.com/leaky_bucket.GIF)

上图可以直观的表现出漏桶算法的思想，请求可以看成水流，以一定速率的出水进入漏桶中，然后漏桶以一定的速率出水，当水流入速度过大会溢出，漏桶算法会强行限制数据的传输速率。突发流量会被限制成一个稳定的流量，如果漏桶溢出，那么数据包或者请求就会被丢弃。漏桶算法的缺陷是不能有效利用系统资源，即使不存在资源冲突，漏桶仍然已恒定的速率流出，故不能应对突发特性的流量。

令牌桶算法的原理是系统已恒定的速率往桶内放入令牌，请求过来需要先从桶内取得令牌然后请求被处理，如果没取到则拒绝服务。假设限制速率2r/s即一秒2个往桶中放入令牌，桶的容量为n,当一个a个字节的数据包到达时，将从桶中删除a个令牌，如果当前桶的令牌数小于a，则本次请求被限流；令牌桶的一个优点在于可以按需改变放入令牌的速率来应对突发性的流量。
![](http://clouder123.oss-cn-beijing.aliyuncs.com/token_bucket.JPG)

## RateLimiter

RateLimiter出自大名鼎鼎的guava包，它是令牌桶算法的一种实现。RateLimiter经常用于限制对一些物理资源或者逻辑资源的访问速率。与Semaphore 相比，Semaphore 限制了并发访问的数量而不是使用速率。
考虑下面的场景：我们需要执行一堆任务，但我们希望任务的提交速率限制在最多1s一个。我们利用RateLimiter来实现。

```
final RateLimiter rateLimiter = RateLimiter.create(1);
Executor threadPool = Excutors.newFixedThreadPool(10);    
 void submitTasks(List tasks) {
        for (Runnable task : tasks) {
            rateLimiter.acquire(); // 需要等待拿到令牌
            executor.execute(task);
        }
  }
```
下面再来考虑一个抢购的场景对接口进行限流

```
@Controller
public class ActivityController {
    @Resource
    private GoodService goodService;
    RateLimiter rateLimiter = RateLimiter.create(10);
    @RequestMapping("/rush")
    public Object rush(HttpServletRequest request) {
        rateLimiter.acquire();
        if (goodService.update(object) > 0) {
            return "success";
        }
        return "fail";
    }
```
RateLimiter用法十分简单，它主要提供了下面方法

* acquire() 从RateLimiter获取一个许可，该方法会被阻塞直到获取到请求并返回等待时间。
* acquire(int permits) 提供一个入参指定许可数。
* create(double permitsPerSecond) 指定每秒多少许可创建RateLimiter。
* setRate(double permitsPerSecond) 根据许可产生速率。
* tryAcquire()/tryAcquire(int permits) 获取许可（指定许可数),非阻塞立即返回，无法获取返回false。

上面列出RateLimiter的几个重要的方法，了解了令牌桶的原理很容易理解成RateLimiter起了一个线程以一个固定的速率给一个计数器比如AtomicInteger加数字，可是真的是这样吗？我们移步代码。

```
public static RateLimiter create(double permitsPerSecond) {
        return create(RateLimiter.SleepingTicker.SYSTEM_TICKER, permitsPerSecond);
    }
    @VisibleForTesting
    static RateLimiter create(RateLimiter.SleepingTicker ticker, double permitsPerSecond) {
        RateLimiter rateLimiter = new RateLimiter.Bursty(ticker, 1.0D);
        rateLimiter.setRate(permitsPerSecond);
        return rateLimiter;
    }    }
```
create方法没什么好说，创建一个RateLimiter,关于这里面的SleepingTicker的作用我们稍后再说。先来看一下acquire方法.

```
public double acquire(int permits) {
        checkPermits(permits);  //参数校验
        Object var4 = this.mutex;
        long microsToWait;
        synchronized(this.mutex) {
        //计算请求需要让线程等待多少时间
            microsToWait = this.reserveNextTicket((double)permits, this.readSafeMicros());
        }
 this.ticker.sleepMicrosUninterruptibly(microsToWait);//阻塞线程
        return 1.0D * (double)microsToWait / (double)TimeUnit.SECONDS.toMicros(1L);
    }
```
下面计算线程的等待时间

```
private long reserveNextTicket(double requiredPermits, long nowMicros) {
        this.resync(nowMicros);  //重新设置下一次获取时间和存储的令牌数
        long microsToNextFreeTicket = this.nextFreeTicketMicros - nowMicros; //需要等待的时间
        double storedPermitsToSpend = Math.min(requiredPermits, this.storedPermits);//获得请求的令牌数和存储的令牌数中的较小值
        double freshPermits = requiredPermits - storedPermitsToSpend;  //如果为0 表示本次请求令牌够用,大于0表示“欠”的令牌数
        long waitMicros = this.storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend) + (long)(freshPermits * this.stableIntervalMicros);  //计算欠的令牌数需要多久产生
        this.nextFreeTicketMicros += waitMicros;  //下次获取的时间
    this.storedPermits -= storedPermitsToSpend; //更新令牌数
        return microsToNextFreeTicket;
    }
    
    ...
    
private void resync(long nowMicros) {
        if(nowMicros > this.nextFreeTicketMicros) {
            this.storedPermits = Math.min(this.maxPermits, this.storedPermits + (double)(nowMicros - this.nextFreeTicketMicros) / this.stableIntervalMicros);
            this.nextFreeTicketMicros = nowMicros;
        }
    }
```
resync的方法作用是设置nextFreeTicketMicros和storedPermits；nextFreeTicketMicros表示下次获取的时间，初始化为0。每调用一次acquire()，nowMicros - nextFreeTicketMicros就是上次请求到这次请求中间发生的时间。如果当前时间比上一轮设置的下次获取的时间大（因为存在提前获取的情况，比如上次直接获取了10个，那上轮设置的nextFreeTicketMicros就是上一轮的时间+5s），那就计算这个中间理论上能生成多少的令牌。比如这中间隔了1秒钟，然后stableIntervalMicros=5000（稳定生成速度的情况下）,那么，就这中间就可以生成2个令牌。再加上它原先存储的storedPermits个，如果比maxPermits大，那最大也只能存maxPermits这么多。如果比maxPermits小，那就是storedPermits=原先存的+这中间生成的数量。同时记录下下次获取的时候需要减去的时间，也就是当前时间 （nextFreeTicketMicros ）。

reserveNextTicket方法计算出的时间也是请求需要等待的时间。举个例子，假设在nowMicros=3这个时间点来了一个请求，需要5个令牌，当前时间点令牌数为2，假设计算出waitMicro=3,则nexFreeTicketMicors=3,当前storedPermits=-2;该请求需要等待3s后的一致性。再假设过了1s另一请求到来需要1个令牌，则根据nexFreeTicketMicors和storedPermits继续计算waitMicro获取阻塞时间。

总结起来，RateLimiter并不是真的一直往某个地方放令牌，而是在每次acquire时候根据当前令牌数和下次获取令牌算去本次请求是否能执行，如果要等需要等待多长时间，设计上非常巧妙。当第一次调用accquire()的时候，resync会被执行，然后在accquire()中将nextFreeTicketMicros设置为当前时间。但是，还可以请求的令牌数和当前存储的令牌数进行比较。如果请求的令牌数很大，则会计算出生成这些多余的令牌需要的时间，并加在nextFreeTicketMicros上，从而保证下次调用accquire()的时候，根据nextFreeTicketMicros和当时的nowMicros相减，若>0，则需要等到对应的时间。也就能应对流量的突增情况了。