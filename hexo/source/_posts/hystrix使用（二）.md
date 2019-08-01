---
title: hystrix使用（二）
categories: hystrix
date: 2017-10-30 11:42:31
tags: hystrix
---

继上一篇介绍的Hystrix的使用之后，本篇内容进一步探讨一下Hytrix的使用，下面直接进入正题。

## 隔离策略
Hytrix应用舱壁模式（货船为了进行防止漏水和火灾的扩散，将货仓分割成多个，以此减小意外带来的损失）来隔离依赖，限制对依赖的并发访问。
Hystrix有两种隔离策略：线程隔离和信号量隔离。

#### 线程隔离
每个外部依赖都在隔离的线程中执行，将这些对外部的调用从调用线程中隔离开，从而不会影响调用者的整个流程。Hystrix底层通过为每个依赖建一个线程池的方式来达到当次依赖的线程池资源耗尽时不会影响其他依赖。

#### 信号量隔离
信号量一般用来控制对给定依赖的并发请求数量。通过使用信号量的方式来控制负载。但是它不允许超时和非阻塞请求。如果足够信任依赖方并且仅仅想限制负载时，可以使用此方法。HytrixCommand和HystrixObservableCommand在两个地方实施信号量隔离：Fallback和Execution。

配置Command为信号隔离还是线程隔离：以下为线程隔离的方式，同时可以设置线程池的属性。

```
public class CommandUsingThreadsolation extends HystrixCommand<String> {
    private final int id;
    public CommandUsingThreadsolation(int id) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                //THREAD : 线程隔离     SEMAPHORE : 信号量隔离
                        .withExecutionIsolationStrategy(ExecutionIsolationStrategy.THREAD))
                .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
                        .withCoreSize(20)
                                .withMaximumSize(50).withMaxQueueSize(65535)
                                .withQueueSizeRejectionThreshold(200) //队列大于此值时抛异常
                        .withKeepAliveTimeMinutes(1)
                                .withMetricsRollingStatisticalWindowInMilliseconds(20000)
                        .withMetricsRollingStatisticalWindowBuckets(20)
                ));
        this.id = id;
    }
    @Override
    protected String run() {
        return "ValueFromHashMap_" + id;
    }
}
```
对于信号量，可以配置如下：

```
public class CommandUsingSemaphoreIsolation extends HystrixCommand<String> {
    private final int id;
    public CommandUsingSemaphoreIsolation(int id) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                                .withExecutionIsolationStrategy(ExecutionIsolationStrategy.SEMAionStrategy.SEMAPHORE)
                        .withExecutionIsolationSemaphoreMaxConcurrentRequests(20)));
        this.id = id;
    }
    @Override
    protected String run() {
        return "ValueFromHashMap_" + id; 
    }
}
```

## 熔断机制

熔断是指在因某些原因导致系统过载时，为了防止其拖垮整个服务而采取的措施。当熔断开关打开是，系统将关闭对某一服务的请求，转而走降级逻辑或向调用者抛错。

Hystrix中实现了断路器逻辑，并为每个commandKey标识的服务创建一个断路器。HystrixCommand和HystrixObservableCommand将断路器作为其属性，作用在命令执行过程中。默认情况下，请求都会走熔断判断逻辑。可以通过circuitBreaker.forceClosed=true来使熔断器失效。或者circuitBreaker.forceOpen=true是的所有请求都走降级逻辑。

断路器共有三个状态：OPEN、CLOSE、HALF-OPEN

* OPEN开启状态下，所有的请求都被降级；
* CLOSE关闭状态下，请求都正常执行；
* HALF-OPEN半开状态，断路器为开启状态一段时建后会放过一个请求，如果这个请求失败则将断路器改为OPEN状态，成功则将断路器设置为CLOSE状态。

下面简单开下Hystirx断路器如何生效的：

Hystrix命令执行完之后，会生成很多计量信息，如命令执行结果类型，指一段时间内，命令执行成功、失败、超时以及被拒绝的数量。
这些度量信息是如何记录的呢？ Hystrix使用滑动窗口的方式来保存度量数据，它只会保存最近时间段（rollingStats.timeInMilliseconds）内的所有命令执行结果信息，并且为了更好地使用它们，会将整个时间段的度量分为rollingStats.numBuckets个桶来保存；断路器执行时，会根据这些数据来计算错误率供断路器使用。

断路器的CLOSE状态向OPEN状态转换时，需要满足以下条件：

* 滑动窗口周期内的总请求数量大于阈值HystrixCommandProperties.circuitBreakerRequestVolumeThreshold();
* 滑动窗口周期内请求错误率超过设置的阈值 HystrixCommandProperties.circuitBreakerErrorThresholdPercentage()

断路器由OPEN -> HALF-OPEN -> CLOSE转换时，需要满足条件：
在断路器开启HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds后，会放过一个请求，此时断路器状态更改为HALF-OPEN状态，此请求成功时，则由HALF-OPEN状态转向CLOSE。配置如下：

```
public class CommandUsingCircuitBreaker extends HystrixCommand< String> {
    private final int id;
    public CommandUsingCircuitBreaker(int id) { super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey( "ExampleGroup"))
            .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                            .withExecutionIsolationStrategy(ExecutionIsolationStrategy.THREAD)
                            .withCircuitBreakerEnabled(true) //熔断器
                            .withCircuitBreakerRequestVolumeThreshold(20) // 熔断器起作用的请求数量阈值
                            .withCircuitBreakerSleepWindowInMilliseconds(500)// 休眠事件
                            .withCircuitBreakerErrorThresholdPercentage (50)// 断路器打开错误的边界
                    .withCircuitBreakerForceClosed(false)
                    .withCircuitBreakerForceOpen(false) )
            );
        this.id = id;
    }
    @Override
    protected String run() {
        return "ValueFromHashMap_" + id; }
}
```

## 请求合并

HystrixCollapser用于将多个请求合并成一个请求发出。将多个请求合并成一个请求的好处是：可以减少线程的数量和网络连接的数量，减少系统代价。HystrixCollapser使用全自动的方式来实现合并请求，而不需要开发人员以硬编码的形式来支持批量请求。

但是它也是有一定的代价：它需要等待一个窗口期（timerDelayInMilliseconds默认10ms）来接受批量请求，然后将请求批量发送。因此对延迟要求低的服务不推荐使用此方法。

```
class CommandCollapserGetValueForKey extends HystrixCollapser<List<String>, String, Integer> {
    private final Integer key;
    private static final Setter setter = Setter.withCollapserKey(HystrixCollapserKey.Factory.asKey("commandCollapserGetValueFo rKey"))
            .andCollapserPropertiesDefaults(HystrixCollapserProperties.Setter()
                    .withMaxRequestsInBatch(20)//最大批量请求数量
                    .withTimerDelayInMilliseconds(5).withRequestCacheEnabled(true)
            ).andScope(Scope.REQUEST);
    public CommandCollapserGetValueForKey(Integer key) {
        super(setter);
        this.key = key;
    }
    @Override
    public Integer getRequestArgument() {
        return key;
    }
    @Override
    protected HystrixCommand<List<String>> createCommand(final Collection<CollapsedRequest<String, Integer>> requests) {
        return new BatchCommand(requests);
    }
    @Override
    protected void mapResponseToRequests(List<String> batchResonse, Collection<CollapsedRequest<String, Integer>> requests) {
        int count = 0;
        for (CollapsedRequest<String, Integer> request : requests) {
            request.setResponse(batchResponse.get(count++));
        }
    }
    private static final class BatchCommand extends HystrixCommand<List<String>> {
        private final Collection<CollapsedRequest<String, Integer>> requests;
        private BatchCommand(Collection<CollapsedRequest<String, Integer>> requests) {
            super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                    .andCommandKey(HystrixCommandKey.Factory.as
                            Key("GetValueForKey")));
            this.requests = requests;
        }
        @Override
        protected List<String> run() {
            ArrayList<String> response = new ArrayList<String>( for (CollapsedRequest<String, Integer> request : re
            response.add("ValueForKey: " + request.getArgum
        }
        return response;
    }
}
}
```

## 注解使用
在前面的示范中可以看出接入Hystrix需要做很多硬编码工作，对代码侵入很大。因此开源社区的贡献者开发了Hystrix的注解接入方式，旨在帮助开发人员提高效率。基于注解的方式需要核外导入依赖。

```
<dependency>
    <groupId>com.netflix.hystrix</groupId>
    <artifactId>hystrix-javanica</artifactId>
    <version>${hystrix.version}</version>
</dependency>
```
hystrix-javanica使用AspectJ来实施切面。所以需要进行如下配置：

```
<aop:aspectj-autoproxy proxy-target-class="true" />
```
### 使用方式

#### 同步方式

```
public class UserService {
    //groupKey默认取Class名, commandKey默认取方法名
    @HystrixCommand(groupKey = "UserGroup", commandKey = "getUserById",
            commandProperties = {
                    @HystrixProperty(name = "execution.isolation.strategy", value = "THREAD"),
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")}
    )
    public User getUserById(String id) {
        return userResource.getUserById(id);
    }
}
```
#### 异步方式
```
public class UserService {
    //groupKey默认取Class名, commandKey默认取方法名
    @HystrixCommand(groupKey = "UserGroup", commandKey = "getUserById")
    public Future<User> getUserByIdAsync(String id) {
        return new AsyncResult<User>() {
            @Override
            public User invoke() {
                return userResource.getUserById(id);
            }
        };
    }
}
```
### Hystrix Collapser
```
@HystrixCollapser(batchMethod = "getUserByIds") 
public Future<User> getUserByIdAsync(String id) {
        return null; 
}
```

## 动态配置
Hystrix默认支持动态配置，在运行时改变参数。它支持以特定的时间周期来执行配置拉取和替换，使用它的动态配置只需要实现PolledConfigurationSource接口，并在其中实现配置拉取的逻辑即可。如下，每3000秒拉取一次更新的配置：

```
public class HystrixDynamicConfigurationManager implements InitializingBean {
    @Resource
    private MtConfigClient mtConfigClient;
    @Override
    public void afterPropertiesSet() throws Exception {
        AbstractConfiguration configInstance = ConfigurationMan
        ager.getConfigInstance();
        AbstractPollingScheduler scheduler = new FixedDelayPoll ingScheduler(0, 3000, true);
        PolledConfigurationSource source = new MtPoolConfigSour ce(mtConfigClient, "hystrixConfig");
        scheduler.setIgnoreDeletesFromSource(true);
        scheduler.startPolling(source, configInstance);
        ConfigurationManager.install(configInstance);
    }
}
```

## Dashboard
Hystrix可以做到实时地监控命令的度量信息，观察命令的执行情况。通过实时观察这些信息，我们可以做实时调整设置，如超时时间、观察到系统的运行情况：延时、超时数量、断路器是否打开等，当系统出现故障时可以第一时间做出应对。关于Dashboard的使用参考官方文档。

####参考
[Hystrix Wiki](https://github.com/Netflix/Hystrix/wiki)

[How-To-Use](https://github.com/Netflix/Hystrix/wiki/How-To-Use)
 
