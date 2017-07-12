---
layout: post
title:  "Guava Cache 分析"
date:  2016-11-05
categories: [cache]
keywords: guava,cache,expireAfterWrite,refreshAfterWrite
---

## guava的单线程回源

缓存的更新有两种方法:

- 被动更新：先从缓存获取，没有则回源获取，再更新缓存；
- 主动更新：发现数据改变后直接更新缓存（在分布式场景下，不容易实现）

在高并发环境，被动回源是需要注意的。
问题：高并发场景下，大量请求在同一时间回源，大量的请求同一时间穿透到后端，容易引起后端服务崩溃（也容易引起并发问题）。

guava cache解决办法：
guava cache保证单线程回源，对于同一个key，只让一个请求回源load，其他线程阻塞等待结果。同时，在Guava里可以通过配置expireAfterAccess/expireAfterWrite设定key的过期时间，key过期后就单线程回源加载并放回缓存。

这样通过Guava Cache简简单单就较为安全地实现了缓存的被动更新操作。

但是如果对于同一时间大量不同的key同时过期，造成大量不同的key同时回源，这种怎么解决呢？

> guava cache实现类似ConcurrentHashMap，维护segment数组，每个segment独享一个锁，ConcurrentHashMap是通过这种机制来实现分段锁，ConcurrentHashMap默认分了16个segment；
> guava Cache默认是4个segment，故guava cache的并发级别默认是4个，也就是说默认情况下，即便是大量不同的key同时过期，最多只也有4个线程并发回源，理论上不会给后端造成过大的压力。

## guava refresh和expire刷新机制

- expireAfterAccess: 当缓存项在指定的时间段内没有被读或写就会被回收。
- expireAfterWrite：当缓存项在指定的时间段内没有更新就会被回收。
- refreshAfterWrite：当缓存项上一次更新操作之后的多久会被刷新。

仅仅使用 expireAfterWrite或者expireAfterAccess就可以实现缓存定时过期，但是频繁的过期会造成频繁的单线程回源，然而guava cache回源的时候会独占一个segment的锁，对于同一个segment的其他的读操作 处于loading状态的则会继续等待，value expire或者为null的key则会阻塞等待segment的锁。

 expireAfterWrite或者expireAfterAccess的实现在数据回源的时候会让请求block住，以获取最新的值。数据实时性保证的较好，但是阻塞住请求对于一些响应要求严苛的业务可能是没办法接受的。那有没有解决的办法呢？

我们且看refreshAfterWrite：

refreshAfterWrite通过定时刷新可以让缓存项保持可用。缓存项只有在被检索时才会真正刷新（如果CacheLoader.refresh实现为异步，那么检索不会被刷新拖慢）。也是保证同一个segment的单线程回源，但是与expireAfterWrite不同的是：其他线程访问loading状态的key时，仅仅稍微等一会，没有等到就返回旧值，整个请求就比较平滑。
与此同时，也引入了一个问题，refreshAfterWrite策略下，如果一个key长期没有被访问，就有可能会访问到很久之前的旧值。例如refreshAfterWrite(5),5s刷新一次，如果1min内，这个key都没有被访问，那么1min之后访问这个key，仍然有可能访问到旧数据，尽管我们设置了5s刷新一次。（guava cache并没单独的线程来处理刷新的逻辑，而是通过读操作来触发清理工作）


对于这个问题有没有这种的办法呢？
guava cache支持我们同时使用expireAfterWrite&refreshAfterWrite，我们既可以通过组合的策略既保证性能，又保证不要读取到太旧的数据。
比如我们有需求：要求请求必须平滑，而且不能读到5s之前的旧数据。

我们可以如下设置来满足需求：

```java
 LoadingCache<String, String> cache = CacheBuilder
            .newBuilder()
            .refreshAfterWrite(4L, TimeUnit.SECONDS)
            .expireAfterWrite(5L, TimeUnit.SECONDS)
            .build(loader);
```

 `expireAfterWrite(5L, TimeUnit.SECONDS)`能保证不会读到5s之前的旧数据，`refreshAfterWrite(4L, TimeUnit.SECONDS)`能保证大部分请求都在4s左右被刷新，小部分访问量较少的在5s的时候通过expireAfterWrite策略重新被回源。


## guava后台异步刷新
refreshAfterWrite的刷新调用的是reload，reload的默认实现是在当前线程里面reload，也会造成一些卡顿，如果希望异步reload，需要重载这个方法。

```java
 @GwtIncompatible // Futures
  public ListenableFuture<V> reload(K key, V oldValue) throws Exception {
    checkNotNull(key);
    checkNotNull(oldValue);
    return Futures.immediateFuture(load(key));//当前线程调用load，会造成当前线程的卡顿，如果不接受卡顿，需要重载这个方法
  }
```