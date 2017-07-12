---
layout: post
title:  "Guava Cache使用示例"
date:  2016-11-05
categories: [cache]
keywords: guava,cache,example,示例
---

## 1.使用场景
>缓存在很多场景下都是相当有用的。例如，计算或检索一个值的代价很高，并且对同样的输入需要不止一次获取值的时候，就应当考虑使用缓存。

>Guava Cache与ConcurrentMap很相似，但也不完全一样。最基本的区别是ConcurrentMap会一直保存所有添加的元素，直到显式地移除。相对地，Guava Cache为了限制内存占用，通常都设定为自动回收元素。在某些场景下，尽管LoadingCache 不回收元素，它也是很有用的，因为它会自动加载缓存。

通常来说，Guava Cache适用于：

- 你愿意消耗一些内存空间来提升速度。
- 你预料到某些键会被查询一次以上。
- 缓存中存放的数据总量不会超出内存容量。（Guava Cache是单个应用运行时的本地缓存。它不把数据存放到文件或外部服务器。如果这不符合你的需求，请尝试Memcached这类工具）

如果你的场景符合上述的每一条，Guava Cache就适合你。

如同范例代码展示的一样，Cache实例通过CacheBuilder生成器模式获取，但是自定义你的缓存才是最有趣的部分。

注：如果你不需要Cache中的特性，使用ConcurrentHashMap有更好的内存效率——但Cache的大多数特性都很难基于旧有的ConcurrentMap复制，甚至根本不可能做到。

## 2.如何使用Guava Cache

下面我们以一个简单的例子开始，缓存大写的字符串key。

```java
 @Test
    public void whenCacheMiss_thenValueIsComputed() throws InterruptedException {
        CacheLoader<String, String> loader;
        loader = new CacheLoader<String, String>() {
            // 当guava cache中不存在，则会调用load方法
            @Override
            public String load(String key) {
                return key.toUpperCase();
            }
        };

        LoadingCache<String, String> cache;
        cache = CacheBuilder
                .newBuilder()
                // 写数据1s后重新加载缓存
                .refreshAfterWrite(1L, TimeUnit.SECONDS)
                .build(loader);

        assertEquals(0, cache.size());
        cache.put("test", "test");
        assertEquals("test", cache.getUnchecked("test"));
        assertEquals("HELLO", cache.getUnchecked("hello"));
        assertEquals(2, cache.size());

        TimeUnit.SECONDS.sleep(2);
        assertEquals("TEST", cache.getUnchecked("test"));

    }
```

## 3.回收策略
所有的cache都需要定期remove value，下面我们看看guava cache的回收策略。

### 3.1. 基于容量的回收(Eviction by Size)
我们可以通过`maximumSize()`方法限制cache的size，如果cache达到了最大限制，oldest items 将会被回收。
```java
 @Test
    public void whenCacheReachMaxSize_thenEviction() {
        CacheLoader<String, String> loader;
        loader = new CacheLoader<String, String>() {
            @Override
            public String load(String key) {
                return key.toUpperCase();
            }
        };
        LoadingCache<String, String> cache;
        cache = CacheBuilder.newBuilder().maximumSize(3).build(loader);

        cache.getUnchecked("first");
        cache.getUnchecked("second");
        cache.getUnchecked("third");
        cache.getUnchecked("forth");
        assertEquals(3, cache.size());
        assertNull(cache.getIfPresent("first"));
        assertEquals("FORTH", cache.getIfPresent("forth"));
    }
```

我们也可以限制cache size通过定制`weight function`，以下为自定义`weight function`实现的限制cache size 的示例：

```java
  @Test
    public void whenCacheReachMaxWeight_thenEviction() {
        CacheLoader<String, String> loader;
        loader = new CacheLoader<String, String>() {
            @Override
            public String load(String key) {
                return key.toUpperCase();
            }
        };

        Weigher<String, String> weighByLength;
        weighByLength = new Weigher<String, String>() {
            @Override
            public int weigh(String key, String value) {
                return value.length();
            }
        };

        LoadingCache<String, String> cache;
        cache = CacheBuilder.newBuilder()
                .maximumWeight(16)
                .weigher(weighByLength)
                .build(loader);

        cache.getUnchecked("first");
        cache.getUnchecked("second");
        cache.getUnchecked("third");
        cache.getUnchecked("last");
        assertEquals(3, cache.size());
        assertNull(cache.getIfPresent("first"));
        assertEquals("LAST", cache.getIfPresent("last"));
    }
```

### 3.2. 定时回收(Eviction by Time)
除了通过size来回收记录，我们也可以选择定时回收

`CacheBuilder`提供两种定时回收的方法：

- `expireAfterAccess(long, TimeUnit)`：缓存项在给定时间内没有被读/写访问，则回收。请注意这种缓存的回收顺序和基于大小回收一样。
- `expireAfterWrite(long, TimeUnit)`：缓存项在给定时间内没有被写访问（创建或覆盖），则回收。如果认为缓存数据总是在固定时候后变得陈旧不可用，这种回收方式是可取的。

下面代码 将示例`expireAfterAccess`的用法：

```java
 @Test
    public void whenEntryIdle_thenEviction() throws InterruptedException {
        CacheLoader<String, String> loader;
        loader = new CacheLoader<String, String>() {
            @Override
            public String load(String key) {
                return key.toUpperCase();
            }
        };

        LoadingCache<String, String> cache;
        cache = CacheBuilder.newBuilder()
                .expireAfterAccess(2, TimeUnit.MILLISECONDS)
                .build(loader);

        cache.getUnchecked("hello");
        assertEquals(1, cache.size());

        cache.getUnchecked("hello");
        Thread.sleep(300);

        cache.getUnchecked("test");
        assertEquals(1, cache.size());
        assertNull(cache.getIfPresent("hello"));
    }
```

### 3.3.基于引用的回收(Reference-based Eviction)

通过使用弱引用的键、或弱引用的值、或软引用的值，Guava Cache可以把缓存设置为允许垃圾回收：

- CacheBuilder.weakKeys()：使用弱引用存储键。当键没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式（==），使用弱引用键的缓存用==而不是equals比较键。
- CacheBuilder.softValues()：使用软引用存储值。软引用只有在响应内存需要时，才按照全局最近最少使用的顺序回收。考虑到使用软引用的性能影响，我们通常建议使用更有性能预测性的缓存大小限定（见上文，基于容量回收）。使用软引用值的缓存同样用==而不是equals比较值。

### 3.4.显式清除
任何时候，你都可以显式地清除缓存项，而不是等到它被回收：

- 个别清除：`Cache.invalidate(key)`
- 批量清除：`Cache.invalidateAll(keys)`
- 清除所有缓存项：`Cache.invalidateAll()`

### 4.移除监听(RemovalNotification)
>通过CacheBuilder.removalListener(RemovalListener)，你可以声明一个监听器，以便缓存项被移除时做一些额外操作。缓存项被移除时，RemovalListener会获取移除通知[RemovalNotification]，其中包含移除原因[RemovalCause]、键和值。

**请注意，RemovalListener抛出的任何异常都会在记录到日志后被丢弃[swallowed]。**

  

```java
 @Test
    public void whenEntryRemovedFromCache_thenNotify() {
        CacheLoader<String, String> loader;
        loader = new CacheLoader<String, String>() {
            @Override
            public String load(final String key) {
                return key.toUpperCase();
            }
        };

        RemovalListener<String, String> listener;
        listener = new RemovalListener<String, String>() {
            @Override
            public void onRemoval(RemovalNotification<String, String> n) {
                if (n.wasEvicted()) {
                    String cause = n.getCause().name();
                    assertEquals(RemovalCause.SIZE.toString(), cause);
                }
            }
        };

        LoadingCache<String, String> cache;
        cache = CacheBuilder.newBuilder()
                .maximumSize(3)
                .removalListener(listener)
                .build(loader);

        cache.getUnchecked("first");
        cache.getUnchecked("second");
        cache.getUnchecked("third");
        cache.getUnchecked("last");
        assertEquals(3, cache.size());
    }
```

## 5.刷新( Refresh the Cache)
刷新和回收不太一样。正如LoadingCache.refresh(K)所声明，刷新表示为键加载新值，这个过程可以是异步的。在刷新操作进行时，缓存仍然可以向其他线程返回旧值，而不像回收操作，读缓存的线程必须等待新值加载完成。

如果刷新过程抛出异常，缓存将保留旧值，而异常会在记录到日志后被丢弃[swallowed]。

重载CacheLoader.reload(K, V)可以扩展刷新时的行为，这个方法允许开发者在计算新值时使用旧的值。

```java
    @Test
    public void cache_reLoad() {
        CacheLoader<String, String> loader;
        loader = new CacheLoader<String, String>() {
            @Override
            public String load(String key) {
                return key.toUpperCase();
            }

            /**
             * 重写reload方法可以定制自己的reload策略
             * @param key
             * @param oldValue
             * @return
             * @throws Exception
             */
            @Override
            public ListenableFuture<String> reload(String key, String oldValue) throws Exception {
                return super.reload(key, oldValue);
            }
        };

        LoadingCache<String, String> cache;
        cache = CacheBuilder.newBuilder()
                .build(loader);
    }

```

`CacheBuilder.refreshAfterWrite(long, TimeUnit)`可以为缓存增加自动定时刷新功能。和`expireAfterWrite`相反，`refreshAfterWrite`通过定时刷新可以让缓存项保持可用，但请注意：缓存项只有在被检索时才会真正刷新（如果CacheLoader.refresh实现为异步，那么检索不会被刷新拖慢）。因此，如果你在缓存上同时声明expireAfterWrite和refreshAfterWrite，缓存并不会因为刷新盲目地定时重置，如果缓存项没有被检索，那刷新就不会真的发生，缓存项在过期时间后也变得可以回收。

```java
 @Test
    public void whenLiveTimeEnd_thenRefresh() {
        CacheLoader<String, String> loader;
        loader = new CacheLoader<String, String>() {
            @Override
            public String load(String key) {
                return key.toUpperCase();
            }
        };

        LoadingCache<String, String> cache;
        cache = CacheBuilder.newBuilder()
                .refreshAfterWrite(1, TimeUnit.MINUTES)
                .build(loader);
    }
```

## 6. 处理空值(Handle null Values)
>实际上Guava整体设计思想就是拒绝null的，很多地方都会执行com.google.common.base.Preconditions.checkNotNull的检查。 

By default, Guava Cache will throw exceptions if you try to load a null value – as it doesn’t make any sense to cache a null.

But if null value means something in your code, then you can make good use of the Optional class as in the following example:
默认情况guava cache将会抛出异常，如果试图加载null value--因为cache null 是没有任何意义的。
但是如果null value 对你的代码而已有一些特殊的含义，你可以尝试用Optional来表达，像下面这个例子：

```java
@Test
    public void whenNullValue_thenOptional() {
        CacheLoader<String, Optional<String>> loader;
        loader = new CacheLoader<String, Optional<String>>() {
            @Override
            public Optional<String> load(String key) {
                return Optional.fromNullable(getSuffix(key));
            }
        };

        LoadingCache<String, Optional<String>> cache;
        cache = CacheBuilder.newBuilder().build(loader);

        assertEquals("txt", cache.getUnchecked("text.txt").get());
        assertFalse(cache.getUnchecked("hello").isPresent());
    }

    private String getSuffix(final String str) {
        int lastIndex = str.lastIndexOf('.');
        if (lastIndex == -1) {
            return null;
        }
        return str.substring(lastIndex + 1);
    }
```


## 7. 统计

`CacheBuilder.recordStats()`用来开启Guava Cache的统计功能。统计打开后，`Cache.stats()`方法会返回`CacheStats`对象以提供如下统计信息：

- hitRate()：缓存命中率；
- averageLoadPenalty()：加载新值的平均时间，单位为纳秒；
- evictionCount()：缓存项被回收的总数，不包括显式清除。

此外，还有其他很多统计信息。这些统计信息对于调整缓存设置是至关重要的，在性能要求高的应用中我们建议密切关注这些数据。

## 8. Notes
1.什么时候用get，什么时候用getUnchecked
官网文档说：
{% note primary %}
If you have defined a CacheLoader that does not declare any checked exceptions then you can perform cache lookups using getUnchecked(K);
however care must be taken not to call getUnchecked on caches whose CacheLoaders declare checked exceptions.
{% endnote %}

字面意思是，如果你的CacheLoader没有定义任何checked Exception，那你可以使用getUnchecked。


