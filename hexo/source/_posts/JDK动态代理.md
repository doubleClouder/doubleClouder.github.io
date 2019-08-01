---
title: JDK动态代理
categories: 设计模式
date: 2017-03-28 20:38:20
tags: 设计模式
---

到动态代理，我们先来说说代理模式。代理模式是Java中一种常见的也是非常重要的设计模式，先来看下定义：当一个对象不适合或者无法直接访问另一对象时，通过一个代理来控制对另一对象的访问，这个代理对象就起到了一个中介的作用。代理模式的主要特征是代理类和委托类实现同一接口。下面就是代理模式最基本的结构图。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/proxy1.png
)

根据这个定义，我们先写一个简单的代理模式：

```
public interface Service {
    void doService();
}
public class ProxyDemo {
    class ServiceImpl implements Service{
        @Override
        public void doService() {
            System.out.println("do some service");
        }
    }
    class Proxy implements Service{
        private Service service;
        public Proxy(Service service){
            this.service = service;
        }
        @Override
        public void doService() {
            System.out.println("do something before");
            service.doService();
            System.out.println("do something after");
        }
    }
    public static void main(String[] args){
        Service service = new ProxyDemo().new ServiceImpl();
        Service proxy = new ProxyDemo().new Proxy(service);
        proxy.doService();
    }
}

```
上面就是代理模式的一种基本实现，我们称之静态代理。它可以在不修改目标对象的前提下，对目标功能进行拓展。所以缺
点也是显而易见的。因为代理对象需要与目标对象实现一样的接口，所以会有很多代理类，而且,一旦接口增加方法,目标对象与代理对象都要维护。动态代理就解决了这些问题。

## JDK动态代理

动态代理有两种，JDK动态代理和cglib动态代理，本文我先来深入探讨一下JDK动态代理。JDK动态代理需要我们实现
InvocationHandler接口，在invoke方法中对目标类的目标方法进行增强（即定义横切逻辑），还是先上代码看下如何使用。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/proxy2.png
)
这就是jdk动态代理的基本用法，实际这就是一个aop了，spring aop也是使用Proxy和InvocationHandler这两个
类来进行方法增强的。通过这种方式我们就可以动态的代理目标对象，可是它是怎么达到效果的呢？知其然，知其所以然。下面就跟下这两个类的源码。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/proxy3.png
)

上面是Proxy#newProxyInstance方法。我们先看下729行，这里调用目标类的构造方法，然后739行，实例化目标类
并将对象返回给上面的MyInvocationHandler。我们再跟进上面的getproxyClass方法看下。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/proxy4.png
)

进入proxyClassCache.get这里先看目标接口的实例是否有缓存，有就直接返回。没有就通过ProxyclassFactory
这个工厂类创建并加入缓存。最近通过ProxyclassFactory的apply方法创建实例。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/proxy5.png
)
中间一些校验的逻辑跳过
![](https://clouder123.oss-cn-beijing.aliyuncs.com/proxy6.png
)
上面的639行：ProxyGenerator#GenerateProxyClass方法才是生成代理类的关键所在，它其实是动态生成字节码并
保存到硬盘。到这里我们也就了解了jdk是怎样动态的生成的代理类了，可是代理类是怎么样调到我们上面的InvocationH
andler的invoke方法的呢？实际上这个生成代理类会传入一个InvocationHandler对象，代理类在覆盖接口的目标方
法中会调用invoke方法。JDK动态代理大致上就是这样一个过程。