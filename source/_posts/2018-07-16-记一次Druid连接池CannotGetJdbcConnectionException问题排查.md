---
layout: post
title:  "记一次Druid连接池CannotGetJdbcConnectionException问题排查"
date:  2018-07-16 22:39:04
type: case
categories: [case]
keywords: druid,CannotGetJdbcConnectionException,hystrix,中断,排查
---


## 写在前面
最近线上数据库连接池报获取连接超时，一开始没怎么注意，后来异常的数量变多了，就找了时间来查查。发现了一个比较有意思的问题，记录下来。

## 异常

```
### Error updating database. Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection; nested exception is com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 2000, active 32, runningSqlCount 1 
```

## 排查思路

从异常信息来看是连接池拿连接超时，这里有两种可能。
1）数据库连接已经用完，应用层获取不到连接；
2）druid连接池连接已经达到maxActive值

下面命令可以查看db连接池的最大值：
```sql
show variables like '%max_connections%';
```
发现这个值其实还挺大，用完的概率不大。
```
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 5000  |
+-----------------+-------+
```

另外，查看网上，数据库连接用完，异常信息应该是Too many connections 类似的信息（这个没有去验证），所以基本上可以排除是db的连接的原因。

所以大概率是druid连接池达到的最大值maxActive，

这个达到最大值有两个可能，第一种是maxActive配置的太小，并发请求太多。第二种就是某种特定的场景有连接泄露。

到这里基本上没有太好的思路去验证到底是哪种原因导致的了。

首先想到的就是maxActive太小，我们配置的是32，在网络抖动的时候，并发量较大的场景，是有可能会耗完连接的。

如果是连接泄露，应该会有相关异常，搜索exception日志发现如下异常：

```
05:22:33 ERROR version_IS_UNDEFINED [89d552dc1f074730bb5c6459bc58101e][hystrix-ADD_BEAN_CHANGELOG_VALIDATOR-4]c.a.d.p.DruidDataSource - recyle error?
java.lang.InterruptedException: null
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1220) ~[na:1.8.0_40]
        at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335) ~[na:1.8.0_40]
        at com.alibaba.druid.pool.DruidDataSource.recycle(DruidDataSource.java:1266) ~[druid-1.0.9.jar:1.0.9]
        at com.alibaba.druid.pool.DruidPooledConnection.recycle(DruidPooledConnection.java:292) [druid-1.0.9.jar:1.0.9]
        at com.alibaba.druid.filter.FilterChainImpl.dataSource_recycle(FilterChainImpl.java:4534) [druid-1.0.9.jar:1.0.9]
        at com.alibaba.druid.filter.stat.StatFilter.dataSource_releaseConnection(StatFilter.java:646) [druid-1.0.9.jar:1.0.9]
        at com.alibaba.druid.filter.FilterChainImpl.dataSource_recycle(FilterChainImpl.java:4530) [druid-1.0.9.jar:1.0.9]
        at com.alibaba.druid.filter.FilterAdapter.dataSource_releaseConnection(FilterAdapter.java:2717) [druid-1.0.9.jar:1.0.9]
        at com.alibaba.druid.filter.FilterChainImpl.dataSource_recycle(FilterChainImpl.java:4530) [druid-1.0.9.jar:1.0.9]
        at com.alibaba.druid.pool.DruidPooledConnection.close(DruidPooledConnection.java:240) [druid-1.0.9.jar:1.0.9]
        at org.springframework.jdbc.datasource.DataSourceUtils.doCloseConnection(DataSourceUtils.java:341) [spring-jdbc-4.3.8.RELEASE.jar:4.3.8.RELEASE]
        at org.springframework.jdbc.datasource.DataSourceUtils.doReleaseConnection(DataSourceUtils.java:328) [spring-jdbc-4.3.8.RELEASE.jar:4.3.8.RELEASE]
        at org.springframework.jdbc.datasource.DataSourceUtils.releaseConnection(DataSourceUtils.java:294) [spring-jdbc-4.3.8.RELEASE.jar:4.3.8.RELEASE]
        at org.mybatis.spring.transaction.SpringManagedTransaction.close(SpringManagedTransaction.java:127) [mybatis-spring-1.2.3.jar:1.2.3]
        at org.apache.ibatis.executor.BaseExecutor.close(BaseExecutor.java:88) [mybatis-3.3.0.jar:3.3.0]
        at org.apache.ibatis.executor.CachingExecutor.close(CachingExecutor.java:63) [mybatis-3.3.0.jar:3.3.0]
        at sun.reflect.GeneratedMethodAccessor106.invoke(Unknown Source) ~[na:na]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_40]
        at java.lang.reflect.Method.invoke(Method.java:497) ~[na:1.8.0_40]
        at org.apache.ibatis.plugin.Plugin.invoke(Plugin.java:63) [mybatis-3.3.0.jar:3.3.0]
        at com.sun.proxy.$Proxy182.close(Unknown Source) [na:na]
        at org.apache.ibatis.session.defaults.DefaultSqlSession.close(DefaultSqlSession.java:236) [mybatis-3.3.0.jar:3.3.0]
        at org.mybatis.spring.SqlSessionUtils.closeSqlSession(SqlSessionUtils.java:195) [mybatis-spring-1.2.3.jar:1.2.3]
        at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:407) [mybatis-spring-1.2.3.jar:1.2.3]
        at com.sun.proxy.$Proxy73.selectOne(Unknown Source) [na:na]
        at org.mybatis.spring.SqlSessionTemplate.selectOne(SqlSessionTemplate.java:165) [mybatis-spring-1.2.3.jar:1.2.3]
        at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:69) [mybatis-3.3.0.jar:3.3.0]
        at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:53) [mybatis-3.3.0.jar:3.3.0]
        at com.sun.proxy.$Proxy119.queryUserChangeAccountLog(Unknown Source) [na:na]
        at manager.hystrix.ChangeLogCommand.getChangeLogEntity(ChangeLogCommand.java:51) [platform_consume_service.jar:na]
        at manager.hystrix.ChangeLogCommand.run(ChangeLogCommand.java:39) [platform_consume_service.jar:na]
        at manager.hystrix.ChangeLogCommand.run(ChangeLogCommand.java:14) [platform_consume_service.jar:na]
        at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:302) [hystrix-core-1.5.12.jar:1.5.12]
        at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:298) [hystrix-core-1.5.12.jar:1.5.12]
        at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:46) [rxjava-1.2.0.jar:1.2.0]
        at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:35) [rxjava-1.2.0.jar:1.2.0]
```
查看源代码，可以发现是由于：
hystrix降级的时候抛出了Interupted异常，导致DruidDataSource.recycle()回收中断。

```java
 final long lastActiveTimeMillis = System.currentTimeMillis();
        lock.lockInterruptibly();// 这里会响应线程中断，hystrix降级会中断当前线程，
        try {
            activeCount--;
            closeCount++;

            putLast(holder, lastActiveTimeMillis);
            recycleCount++;
        } finally {
            lock.unlock();
        }
    } catch (Throwable e) {
        holder.clearStatementCache();

        if (!holder.isDiscard()) {
            this.discardConnection(physicalConnection);//抛弃连接，不进行回收，而是抛弃
            holder.setDiscard(true);
        }

        LOG.error("recyle error", e);
        recycleErrorCount.incrementAndGet();
    }
```
乍一看，貌似是这里的问题，在回收程序里面出现中断，导致回收逻辑没有处理完。
但是，这个不一定是根本原因，继续看，catch里面的代码，druid将这个hold给Discard。也就是回收逻辑有一层catch保护。
继续看`discardConnection`做了哪些工作？

```java
public void discardConnection(Connection realConnection) {
        JdbcUtils.close(realConnection);//关掉连接

        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            LOG.error("interrupt");
            return;
        }
        try {
            activeCount--;
            discardCount++;

            if (activeCount <= 0) {
                emptySignal();
            }
        } finally {
            lock.unlock();
        }
    }
```
这里可以清晰的看到，这个连接会被close掉，然后抛弃。所以这个连接回收不存在连接泄露。

尽管没有连接泄露，但是这个连接没有正常回收，被抛弃了。总归还不是太好。毕竟创建连接也挺耗。

所以，怎么解决由hystirx线程中断导致druid连接回收异常的问题呢？



###  解决方案
#### hystrix降级但是不中断当前线程
hystrix设置：
execution.isolation.thread.interruptOnTimeout（是否打开超时线程中断）设置false。

能够避免druid因为线程中断造成连接回收失败造成的问题。但是不是最好的方式。
毕竟已经降级，继续执行当前线程其实是浪费资源。

#### 升级druid版本
这个问题的根本原因还是因为druid 实现的原因，不理解为什么druid要调用Lock.lockInterruptibly()这个方法来加锁，

```java
  public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```

释放连接的逻辑应该是要保证一定能释放，所以理论上是要直接调用Lock.lock()方法，不接受中断就好了。
当然最新的版本上已经是这么实现了。
所以升级到最新的版本就好了。


## 总结

由于hystrix降级线程中断，导致druid连接回收异常并不会造成连接泄露，所以大量的`GetConnectionTimeoutException`很可能不是这个原因造成的。

可能是网络抖动造成sql时延上涨，从而导致连接不够用；
也可能是db抖动导致时延上涨，

上面的原因一时间都不太好排查，目前先解决已经发现的问题，后面持续跟踪。  
1）调大连接池数量，这个一定程度上可以缓解问题，减少异常数量  
2）修复hystrix 中断导致连接回收异常


