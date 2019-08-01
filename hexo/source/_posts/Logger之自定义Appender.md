---
title: Logger之自定义Appender
categories: java
date: 2017-03-16 18:11:17
tags: java
---

   如果现在要将应用中所有的Error级别日志收集到数据库中，你能想到什么办法？比较自然的想法是：1. 在所有输出Error日志的地方增加插入数据库操作。2.开辟程序扫描日志文件，遇到Error日志记录入库。这两种方案前者对代码侵入很大；后者工作量较大，而且无法做到实时同步。那么有没有更简便优雅的方式呢？答案就是本文的标题。现在市面上流行的日志框架主要为log4j和logback,二者实现过程基本相同，我们以logback为例，看如何实现自定义Appender，log4j大致类似。
Log4j和Logback主要有三个组件：

* Logger: 负责客户端代码调用，执行error,debug,info,warn等方法。
* Appender: 负责日志的输出，Log4j和Logback都实现了多种不同目标的输出方式，常见的有控制台输出，日志文件输出，socket输出等。
* Layout： 负责日志的格式化。

下面是应用中logback具体的配置：

![](http://clouder123.oss-cn-beijing.aliyuncs.com/123.png)
log4j和logback中总存在rootLogger,其他的logger都继承这个rootLogger。logger中的层次用”.”来分隔，比如
Logger logger = Logger.getLogger(“A.B”) //这里会新建两个logger实例”A”,”A.B”。”A.B”和”A”是一种继承关系，子logger会父logger及祖辈logger的appender添加到自己logger列表，当然我们可以通过配置打破这种继承关系，这里不做赘述。稍微了解了logback的这种结构，那么自定义Appender的问题就迎刃而解了，我们直接上代码。
![](http://clouder123.oss-cn-beijing.aliyuncs.com/124.png)
我们只要在应用启动的时候注入这样一个类，并调用addLogbackAppender()方法，就可以为所有的logger加上一个自定义appender,这样在应用每次输出Error级别日志，就往数据库中记录。当然并不一定是Error日志，具体的逻辑根据需求定义在getCustomizedLogbackAppender中就万事大吉。

