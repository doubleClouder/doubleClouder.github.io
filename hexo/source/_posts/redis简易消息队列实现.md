---
title: redis简易消息队列实现
categories: java
date: 2017-04-08 15:28:51
tags: java
---

大家都知道redis相较同类产品如memcache而言有着丰富的数据类型,这里我们稍微说一下redis中的list数据类型，list的容量是2^32-1个元素，它存储的是链表结构，链表的属性是不管整个链表有多大，它的头尾操作是非常快的，这就符合一个消息队列的特性，简单意义上的消息队列就是生产者从对头插数据，消费者从队尾取数据，所以我们可以用redis中的list来实现一个mq。但是要使用mq，用市面上成熟的消息中间件产品岂不是更好？如rabbitmq、kafka等，当然，如rabbitmq、kafka等，当然如果要保证高可用高性能高吞吐量的消息队列，自然是要考虑使用此类成熟的中间件，这也不在本文描述范畴。但redis作为消息队列的好处是可以对消息的收发做到一个灵活的控制，添加一些自定义的行为，比如给消息定义优先级，所以在一定的业务场景下，将redis作为消息队列不失为一种好的选择。那么到底要如何实现呢。下面我们就演示一下在spring中如何整合redis消息队列。

## 生产者
```
public class RedisMq {
    @Resource
    protected RedisTemplate jedisTemplate;
    public void sendToRedis(final String msg) {
        String listName = "xxx";
        jedisTemplate.opsForList().rightPush(listName, mag);
    }
}
```
生产者的代码很简单，就是往redis的指定list的一端push一条消息。下面就看一下消费者一端如何想spring整合其他mq那样监听队列。

## 消费者
初始化队列类

```
public class InitializeQueue implements InitializingBean, DisposableBean {
    private QueueListener listener;
    private RedisTemplate redisTemplate;
    private static List<RedisQueue> listQ = new ArrayList<>();
    /**
     * 创建启动redis连接并监听
     **
    public void createQueue() {
        //这里先获取所有要监听的redis list名
        List keys = xxx;
        if (keys != null) {
            for (String key : keys) {
                RedisQueue queue = new RedisQueue();
                queue.setListener(listener);
                queue.setRedisTemplate(redisTemplate);
                queue.setKey(app.getAppName());
                listQ.add(queue);
                try {
                    queue.start();
                } catch (Exception e) {
                    log.error("启动redis异常!{}", e);
                }
            }
        }
    }
    public QueueListener getListener() {
        return listener;
    }
    public void setListener(QueueListener listener) {
        this.listener = listener;
    }
    public RedisTemplate getRedisTemplate() {
        return redisTemplate;
    }
    public void setRedisTemplate(RedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    /**
     * 销毁redis连接
     *
     * @throws Exception
     */
    @Override
    public void destroy() throws Exception {
        for (RedisQueue queue : listQ) {
            queue.destroy();
        }
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        //创建队列
        createQueue();
    }
}
```
队列监听器

```
public interface QueueListener<T> {
    public void onMessage(T message);
}
public class RedisQueueListener implements QueueListener<String> {
    private ExecutorService threadpool = Executors.newFixedThreadPool(10);
    
    @Override
    public void onMessage(String value) {
        threadpool.execute(new MessageRunnable(value));
    }
    private class MessageRunnable implements Runnable {
        private String msg
        public MessageRunnable(String msg) {
            this.msg = msg;
        }
        @Override
        public void run() {
            log.info("处理消息!");
            //消息处理业务代码
            ...
        }
    }
}
```
消息队列实例，当创建一个RedisQueue实例就会绑定redis中一个list，并一直从list的尾部取出数据。

```
public class RedisQueue<T> {
    private RedisTemplate redisTemplate;
    private String key;
    private int cap = Short.MAX_VALUE;//最大阻塞的容量，超过容量将会导致清空旧数据
    private byte[] rawKey;
    private RedisConnectionFactory factory;
    private RedisConnection connection;//for blocking
    private BoundListOperations<String, T> listOperations;//noblocking
    private Lock lock = new ReentrantLock();//基于底层IO阻塞考虑
    private QueueListener listener;//异步回调
    private Thread listenerThread;
    private boolean isClosed = false;
    //组塞10分钟
    private int  timeout = 600;
    //休息时间
    private int sleepTime = 1000;
    //获取为null 次数
    private int failCount = 30;
    public void setRedisTemplate(RedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    public void setListener(QueueListener listener) {
        this.listener = listener;
    }
    public void setKey(String key) {
        this.key = key;
    }
    /**
     * 启动监听
     * @throws Exception
     */
    public void start() throws Exception {
        factory = redisTemplate.getConnectionFactory();
        connection = RedisConnectionUtils.getConnection(factory);
        rawKey = redisTemplate.getKeySerializer().serialize(key);
        listOperations = redisTemplate.boundListOps(key);
        if(listener != null){
            listenerThread = new ListenerThread();
            listenerThread.setDaemon(false);
            listenerThread.start();
        }
    }
    /**
     * blocking
     * 队列尾部取数据
     * @return
     */
    public T takeFromTail(int timeout) throws InterruptedException{
        lock.lockInterruptibly();
        try{
            List<byte[]> results = connection.bRPop(timeout, rawKey);
            if(CollectionUtils.isEmpty(results)){
                return null;
            }
            return (T)redisTemplate.getValueSerializer().deserialize(results.get(1));
        }finally{
            lock.unlock();
        }
    }
    public T takeFromTail() throws InterruptedException{
        return takeFromTail(timeout);
    }
    /**
     * 从队列的头，插入
     */
    public void pushFromHead(T value){
        listOperations.leftPush(value);
    }
    public void pushFromTail(T value){
        listOperations.rightPush(value);
    }
    /**
     * 队尾取出
     */
    public T removeFromHead(){
        return listOperations.leftPop();
    }
    public T removeFromTail(){
        return listOperations.rightPop();
    }
    /**
     * 队尾取出增加超时时间
     */
    public T takeFromHead(int timeout) throws InterruptedException{
        lock.lockInterruptibly();
        try{
            List<byte[]> results = connection.bLPop(timeout, rawKey);
            if(CollectionUtils.isEmpty(results)){
                return null;
            }
            return (T)redisTemplate.getValueSerializer().deserialize(results.get(1));
        }catch (Exception e){
            System.out.println(e);
            log.error("get the value fail!",e);
            return null;
        }
        finally{
            lock.unlock();
        }
    }
    public T takeFromHead() throws InterruptedException{
        return takeFromHead(timeout);
    }
    public void destroy() throws Exception {
        if(isClosed){
            return;
        }
        shutdown();
        RedisConnectionUtils.releaseConnection(connection, factory);
    }
    private void shutdown(){
        try{
            listenerThread.interrupt();
        }catch(Exception e){
            //注销失败
            log.error("注销失败!",e);
        }
    }
    class ListenerThread extends Thread {
        int count = 0;
        @Override
        public void run(){
            try{
                while(true){
                    //超过一定次数 休眠一段时间再去获取信息
                    if(count > failCount){
                        Thread.sleep(sleepTime);
                        count = 0;
                    }
                    log.info("开始消费!");
                    T value = takeFromHead();
                    log.info((String)value);
                    //逐个执行
                    if(value != null){
                        try{
                            listener.onMessage(value);
                        }catch(Exception e){
                            log.error("处理消息错误!",e);
                        }
                    }else{
                        count ++;
                    }
                }
            }catch(Exception e){
                //处理消息错误
                log.error("线程获取消息错误!",e);
            }
        }
    }
    public boolean equals(Object obj) {
        if (obj instanceof RedisQueue) {
            RedisQueue queue = (RedisQueue) obj;
            return queue.key.equals(this.key);
        }
        return super.equals(obj);
    }
}
```
最后在Spring上下文中将注入，大功告成。

```
<bean id="initializeQueue" class="com.clouder.redis.InitializeQueue">
        <property name="listener" ref="redisQueueListener"></property>
        <property name="redisTemplate" ref="redisTemplate"></property>
    </bean>
```
上述就是spring中整合redis list实现消息队列的过程。总结一下，创建一个队列初始化类注入spring上下文中随spring启动创建实例，初始化类中根据要监听的队列名创建队列实例，RedisQueue实例中启动后台线程从redis list中取出消息调用redis监听器的onMessage方法。




