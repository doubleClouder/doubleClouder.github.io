---
title: hystrix使用（一）
categories: hystrix
date: 2017-10-29 11:18:53
tags: hystrix
---

继上一篇大致介绍了一下hystrix的设计原则之后，这一篇则重点讲一下Hystrix的使用，毕竟学以致用才是王道！不多废话，直接进入正题。

## 使用
Hystrix的使用有两种方式：原生和注解。不管哪种，首先要引用hystrix的依赖。hystrix的版本参考： [CHANG LOG](https://github.com/Netflix/Hystrix/blob/master/CHANGELOG.md)

```
<dependency>
        <groupId>com.netflix.hystrix</groupId>
        <artifactId>hystrix-core</artifactId>
        <version>${hystrix.version}</version>
</dependency>
```
Hystrix提供了非常便捷的封装——两个可继承的命令抽象HystrixCommand和HystrixObservableCommand。我们要做的就是继承它们，实现run()/construct()方法，在方法中实现对需要隔离的外部依赖和逻辑，剩下的Hystrix就会帮我们完成。

HystrixCommand对于不同的场景提供四个了四个方法：

* execute():以同步阻塞方式执行run();
* queue(): 异步非阻塞方式执行run();
* observe():事件注册前执行run();
* toObservable():事件注册后执行run()。调用toObservable()会直接返回一个Observable对象，而不去执行。当执行subcribe时，才会去执行逻辑run()。

而HystrixObservableCommand则只有observe()和toObservable方法。不同地方在于：HystrixCommand会创建新线程非阻塞执行run(),而HystrixObservableCommand则会在调用程序里执行construct()。

下面来看一个Hystrix官方给出的例子:

```
package com.example.demo;
/**
 * Created by zhusheng02 .
 */
public class CommandHelloWorld extends HystrixCommand<String> {
    private final String name;
    public CommandHelloWorld(String name) {
        Setter setter = Setter.withGroupKey(
                HystrixCommandGroupKey.Factory.asKey("helloWorldGroup")
        )
                .andCommandKey(HystrixCommandKey.Factory.asKey("helloWorldQuery")
                                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("helloWorldThreadPollKey"))
                        ies.Setter()
                                .andCommandPropertiesDefaults(HystrixCommandPropert
                                        .withExecutionIsolationStrategy(THREAD))//SEMAPHORE
                );
        super(setter);
        this.name = name;
    }
    @Override
    protected String run() {
//实现业务逻辑
        System.out.println("Time:" + System.currentTimeMillis() + " Thread:" + Thread.currentThread().getName());
        return "Hello " + name + "!";
    }
    public static class UnitTest {
        // 同步模式访问
        @Test
        public void testSynchronous() {
            assertEquals("Hello World!", new CommandHelloWorld("World").execute());
            assertEquals("Hello Bob!", new CommandHelloWorld("B ob").execute());
        }
        // 异步阻塞
        @Test
        public void testAsynchronous1() throws Exception {
            assertEquals("Hello World!", new CommandHelloWorld("World").queue().get());
            assertEquals("Hello Bob!", new CommandHelloWorld("B ob").queue().get());
        }
        // 异步非阻塞
        @Test
        public void testAsynchronous2() throws Exception {
            d ").queue();
            queue();
            Future<String> fWorld = new CommandHelloWorld("Worl Future<String> fBob = new CommandHelloWorld("Bob").
                    assertEquals("Hello World!", fWorld.get());
            assertEquals("Hello Bob!", fBob.get());
        }
        //监听模式
        @Test
        public void testObservable() throws Exception {
            Observable<String> fWorld = new CommandHelloWorld(" World").observe();
            Observable<String> fBob = new CommandHelloWorld("Bo b").observe();
            ngle());
            ());
            System.out.println("This Step");
// 阻塞
            assertEquals("Hello World!", fWorld.toBlocking().si
                    assertEquals("Hello Bob!", fBob.toBlocking().single
// 非阻塞模式
                            fWorld.subscribe(new Observer<String>() {
                                @Override
                                public void onCompleted() {
                                }
                                @Override
                                public void onError(Throwable e) {
                                }
                                @Override
                                public void onNext(String v) {
                                    System.out.println("onNext: " + v);
                                }
                            });
/*
Time: 1501830396399, Thread: hystrix-ExampleGroup-7 This Step
*/
// 非阻塞模式，省略了异常和onCompleted
            fBob.subscribe(new Action1<String>() {
                @Override
                public void call(String v) {
                    System.out.println("onNext: " + v);
                }
            });
        }
        @Test
        public void toObserables() {
            Observable<String> kitty = new CommandHelloWorld("K itty").toObservable();
            System.out.println("This Step");
            kitty.subscribe(new Action1<String>() {
                @Override
                public void call(String s) {
                    System.out.printf(s);
                }
            });
        }
//  输出结果
//This Step
//Time: 1501830396400, Thread: hystrix-ExampleGroup-8 //Hello Kitty!
    }
}
```
HystrixObservableCommand与HystrixCommand具体不同的是：

* 最直观的差别是前者业务逻辑放在run()里，并直接返回结果；后者的命令逻辑写在construct()里返回Observable;
* 前者默认是线程隔离，后者是信号量隔离。差别后面分析；
* 前者一个实例只能向调用程序发送单条数据；后者可以发送多条数据。

## fallback
降级是指在系统出现问题时，系统能提供一些备用的方法，常见的比如说返回一些默认值或者执行一些降级逻辑;Hystrix优雅的支持降级，只需要再命令类中添加降级方法getFallback()或者resumeWithFallback()。在主命令失败时，可以返回一些默认的结果。

在上面的CommandHelloWorld和ObservableCommandHelloWorld中，可以添加如下的降级方法：

```
//CommandHelloWorld
@Override
protected String getFallback() {
return StringUtils.EMPTY; }
//ObservableCommandHelloWorld
 protected Observable<String> resumeWithFallback() { return Observable.empty();
}
```
Hystrix在什么情况下认为需要降级呢？当发生如下几种情况时，会调用降级逻辑：

* 非HystrixBadRequestException异常，业务逻辑抛出的异常。对于HystrixBadRequestException,其可以被用来包装如参数非法或者非业务逻辑错误等异常，且不会被记录到失败的度量数据中。
* 业务逻辑超时异常。
* 熔断器打开时。
* 线程池满或者信号量满的情况下。

## 请求缓存
Hystrix支持在同一上下文中缓存请求结果。当多次对相同key的资源请求时，只会在第一次真正发起请求，后面的都从缓存得到，此处上下文可以是一次客户端请求上下文。只需要实现getCacheKey()。

```
public class CommandUsingRequestCache extends HystrixCommand<Boolean> {
    private final int value;
    private static final Setter setter = Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup")).
            andCommandKey(HystrixCommandKey.Factory.asKey("commandUsingRequestCache"))
           .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                                    .withExecutionIsolationStrategy(THREAD)
                                    .withRequestCacheEnabled(true));//配置开启请求缓存
    protected CommandUsingRequestCache(int value) {
        super(setter);
        this.value = value;
    }
    @Override
    protected Boolean run() {
        return value == 0 || value % 2 == 0;
    }
    @Override
    protected String getCacheKey() {
        return String.valueOf(value);
    }
    public static class UnitTest {
        @Test
        public void testWithCacheHits() {
            HystrixRequestContext context = HystrixRequestContext.initializeContext();
            try {
                CommandUsingRequestCache command2a = new CommandUsingRequestCache(2);
                CommandUsingRequestCache command2b = new CommandUsingRequestCache(2);
                //第一次发出请求
                assertTrue(command2a.execute());
                assertFalse(command2a.isResponseFromCache());
                //第二次直接从缓存获取
                assertTrue(command2b.execute());
                assertTrue(command2b.isResponseFromCache());
            } finally {
                context.shutdown();
            }
            //开启一个新的上下文
            context = HystrixRequestContext.initializeContext();
            try {
                CommandUsingRequestCache command3b = new CommandUsingRequestCache(2);
                assertTrue(command3b.execute());
                //此时会重新发起一次请求
                assertFalse(command3b.isResponseFromCache()); } finally {
            }finally { context.shutdown();
            }
        }
    }
}
```
对于web服务，我们可以在Filter中建立上下文，请求结束时，关闭上下文。

```
public class HystrixContextFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HystrixRequestContext context = HystrixRequestContext.initializeContext(); 
        try {
            filterChain.doFilter(servletRequest, servletResponse);
        }finally {
            context.shutdown();
        }
    }
    @Override
    public void destroy() { }
}
```

## 总结
本篇大致介绍了一下Hystrix的简单实用，通过实现HystrixCommand的run()方法或者HystrixObservableCommand的construct()方法，它们的区别是一个是线程隔离，一个是信号量隔离，至于两种隔离策略的区别我们留在下一篇继续分析；还大致介绍了一下如何方便的自动降级和使用请求缓存。下一篇我们继续探讨一下Hytrix其他特性的使用。




 

