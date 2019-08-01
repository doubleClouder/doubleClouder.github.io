---
title: mybatis缓存体系
date: 2017-08-27 14:34:17
tags:
---
## Mybatis基本概念

每当我们使用MyBatis开启一次和数据库的会话，MyBatis会创建出一个SqlSession对象表示一次数据库会话。我们先来看下sqlSession的创建过程:

![](https://clouder123.oss-cn-beijing.aliyuncs.com/mybatis1.png)

关于mybatis的初始化过程，这里不做探讨，只看下mybatis对数据库的会话过程中涉及的核心对象。

* Configuration mybatis配置信息（mybatis初始化过程实际就是创建Configuration对象的过程）
* SqlSession 数据库会话
* SqlSessionFactory 数据库会话创建工厂
* Excutor 执行器，调度核心
* StatementHandler 封装了JDBC Statement操作，如设置参数、将Statement结果集转换成List集合。
* MappedStatement 一个节点的抽象。
* BoundSql 动态生成的SQL语句以及相应的参数信息。

## 一级缓存

Mybatis的缓存体系由一级缓存和二级缓存构成，我们先来分析下一级缓存。缓存的意义是显而易见的，可以提高响应速度和降低资源消耗，Mybatis一级缓存的设计初衷也是基于这点。在对数据库的一次会话中，我们有可能会反复地执行完全相同的查询语句，如果不采取一些措施的话，每一次查询都会查询一次数据库,而我们在极短的时间内做了完全相同的查询，那么它们的结果极有可能完全相同，由于查询一次数据库的代价很大，这有可能造成很大的资源浪费。 为了解决这一问题，减少资源的浪费，MyBatis在SqlSession对象中建立一个简单的缓存，将每次查询到的结果结果缓存起来，当下次查询的时候，如果判断先前有个完全一样的查询，会直接从缓存中直接将结果取出，返回给用户，不需要再进行一次数据库查询了。

mybatis执行数据curd操作是通过sqlSession完成。以查询为例，sqlSession的select操作最终交给执行器Excutor来完成，下面看下Excutor的query方法。

```
@Override
 public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
   //得到绑定sql
   BoundSql boundSql = ms.getBoundSql(parameter);
   //创建缓存Key
   CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
   //查询
   return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
 @SuppressWarnings("unchecked")
 @Override
 public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
   ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
   //如果已经关闭，报错
   if (closed) {
     throw new ExecutorException("Executor was closed.");
   }
   //先清局部缓存，再查询。但仅查询堆栈为0，才清。为了处理递归调用
   if (queryStack == 0 && ms.isFlushCacheRequired()) {
     clearLocalCache();
   }
   List<E> list;
   try {
     //加一,这样递归调用到上面的时候就不会再清局部缓存了
     queryStack++;
     //先根据cachekey从localCache去查
     list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
     if (list != null) {
       //若查到localCache缓存，处理localOutputParameterCache
       handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
     } else {
       //从数据库查
       list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
     }
   } finally {
     //清空堆栈
     queryStack--;
   }
   if (queryStack == 0) {
     //延迟加载队列中所有元素
     for (DeferredLoad deferredLoad : deferredLoads) {
       deferredLoad.load();
     }
     // issue #601
     //清空延迟加载队列
     deferredLoads.clear();
     if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
       //如果是STATEMENT，清本地缓存
       clearLocalCache();
     }
   }
   return list;
 }
 
//从数据库查
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  //先向缓存中放入占位符
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    //最后删除占位符
    localCache.removeObject(key);
  }
  //加入缓存
  localCache.putObject(key, list);
  //如果是存储过程，参数也加入缓存
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```
可以发现query方法中首先会通过createCacheKey方法创建缓存key，得到的结果就是一级缓存中的键值。Mybatis对于其缓存key的生成规则： mappedStementId + offset + limit + SQL + queryParams + environment生成hashCode。，CacheKey会根据这些条件来区分每一个CacheKey。cacheKey中会保存key的hashCode,参数个数以及将参数保存在updateList中。可以发现，mybaties根据标签所在的Mapper的Namespace加上标签的id属性、处理分页类RowBounds的limit和offset、sql语句、sql参数这几个条件来判断两次查询是否相同。

通过上面的代码我们发现一级缓存就是执行器BaseExcutor的一个成员变量。

```
protected PerpetualCache localCache;
```
进入PerpetualCache可以揭开一级缓存的真面目，实际上就是一个本地缓存hashMap.

```
public class PerpetualCache implements Cache {
  private String id;
  private Map<Object, Object> cache = new HashMap<Object, Object>();
  ...
```
至此我们可以发现Mybatis一级缓存以单纯的HashMap做缓存，没有容量控制，是一个粗粒度的缓存，没有更新缓存和缓存过期的概念。只适用于一次SqlSession，而一次SqlSession中通常来说并不会有大量的查询操作，而且只要执行update操作（update、insert、delete），都会将这个SqlSession对象中对应的一级缓存清空掉，所以一般情况下不会出现缓存过大，影响JVM内存空间的问题；

```
//SqlSession.update/insert/delete会调用此方法
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  //update操作会清除所有缓存
  clearLocalCache();
  return doUpdate(ms, parameter);
}
...
@Override
  public void clearLocalCache() {
    if (!closed) {
      localCache.clear();
      localOutputParameterCache.clear();
    }
  }
```
总结下一级缓存存储过程：

1. 对于某个Select Statement，根据该Statement生成cacheKey。
2. 判断在Local Cache中,该key是否用对应的数据存在。
3. 如果命中，则跳过查询数据库，继续往下走。
4. 如果没命中,去数据库中查询数据，得到查询结果,将cacheKey和查询到的结果作为key和value，放入Local Cache中,将查询结果返回。
5. 判断缓存级别是否为STATEMENT级别，如果是的话，清空本地缓存。

Mybatis一级缓存的生命周期和SqlSession一致。本质就是一个hashMap,而且不同sqlSession的更新数据会引发脏数据问题，所以建议不使用一级缓存。三种方式不走一级缓存：

1. 一级缓存的默认级别设定为Statement，即不使用一级缓存。
2. 标签中的flushCache属性设置为true
3. Mybatis的拦截器

## 二级缓存

![](https://clouder123.oss-cn-beijing.aliyuncs.com/mybatis2.png
)
在mybatis中，缓存的功能由根接口Cache定义，整个体系采用装饰器模式。数据存储和缓存的基本功能由perpetualCache实现，然后通过一系列的装饰器对perpetualCache进行缓存策略的方便控制。

1. FifoCache：先进先出算法，缓存回收策略
2. LoggingCache：输出缓存命中的日志信息
3. LruCache：最近最少使用算法，缓存回收策略
4. ScheduledCache：调度缓存，负责定时清空缓存
5. SerializedCache：缓存序列化和反序列化存储
6. SoftCache：基于软引用实现的缓存管理策略
7. SynchronizedCache：同步的缓存装饰器，用于防止多线程并发访问
8. WeakCache：基于弱引用实现的缓存管理策略
9. 特殊的装饰器TransactionalCache：事务性的缓存

二级缓存默认关闭。若开启先在Configuration下增加 \，然后在mapper映射文件增加或节点 注解方式增加@CacheNamespace或者@CacheNamespaceRef()。

和cache体系差不多，mybatis执行器也是通过委托的方式实现执行器的灵活控制。

![](https://clouder123.oss-cn-beijing.aliyuncs.com/mybatis3.png
)
若二级缓存开启，则mybatis默认的执行器为CachingExecutor。
流程为： 从二级缓存中进行查询 -> [如果缓存中没有，委托给 BaseExecutor] -> 进入一级缓存中查询 -> [如果也没有] -> 则执行 JDBC 查询。

```
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
  Cache cache = ms.getCache();
  //只有通过<cache/>,<cache-ref/>或@CacheNamespace,@CacheNamespaceRef标记使用缓存的Mapper.xml或Mapper接口才会有二级缓存。
  if (cache != null) {
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, parameterObject, boundSql);
      @SuppressWarnings("unchecked")
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // 
      }
      return list;
    }
  }
  return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```
这里的tcm.putObject方法执行完之后缓存并没有真正的生效，这里只是记录了这次查询将要产生缓存变更，这时候相同的sql查询缓存是不会生效的。下面看下TransactionalCacheManager#putObject（）;

```
public void putObject(Cache cache, CacheKey key, Object value) {
  getTransactionalCache(cache).putObject(key, value);
}
private TransactionalCache getTransactionalCache(Cache cache) {
  TransactionalCache txCache = transactionalCaches.get(cache); 
  if (txCache == null) {
    txCache = new TransactionalCache(cache);
    transactionalCaches.put(cache, txCache);
  }
  return txCache;
}
```
实际只有执行了sqlSession的commit方法之后，缓存的变更会真正的被刷新到缓存中去，开始真正的发挥作用。因为sqlSession的commit会调到上面TransactionalCache的commit。

```
public void commit() {
  if (clearOnCommit) {
    delegate.clear();
  }
  flushPendingEntries();
  reset();
}
public void rollback() {
  unlockMissedEntries();
  reset();
}
private void reset() {
  clearOnCommit = false;
  entriesToAddOnCommit.clear();
  entriesMissedInCache.clear();
}
private void flushPendingEntries() {
  for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
    delegate.putObject(entry.getKey(), entry.getValue());
  }
  for (Object entry : entriesMissedInCache) {
    if (!entriesToAddOnCommit.containsKey(entry)) {
      delegate.putObject(entry, null);
    }
  }
}
```
同样，写操作也不是马上会清除缓存。

```
@Override
 public int update(MappedStatement ms, Object parameterObject) throws SQLException {
//默认刷新缓存
   flushCacheIfRequired(ms);
   return delegate.update(ms, parameterObject);
 }
private void flushCacheIfRequired(MappedStatement ms) {
  Cache cache = ms.getCache();
  if (cache != null && ms.isFlushCacheRequired()) {      
    tcm.clear(cache);
  }
}
```

## 总结

1. 二级缓存是以namespace为单位，不同namespace下互不影响。
2. 写操作会清空所在namespace下所有缓存。
3. 一个namespace下操作多张表会引起脏数据问题。
4. 正是由于Mybatis cache体制这种灵活的委托机制，我们可以借助外部缓存来自定义Mybatis的二级缓存。

上面就是Mybatis的缓存体系，设计上很简单，也存在不少的坑，所以我们使用的时候还是自己实现或者借助外部缓存，让Mybatis只完成它的本职工作就好！



