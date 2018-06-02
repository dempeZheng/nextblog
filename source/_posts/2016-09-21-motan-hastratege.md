---
layout: post
title:  "Motan源码解读-容错策略"
date:  2016-09-21
categories: [Motan源码解读]
keywords: motan,源码,hastratege,容灾策略
---

## Motan的容错策略

>Motan 在集群调用失败时，提供了两种容错方案，并支持自定义扩展。 高可用集群容错策略在Client端生效，因此需在Client端添加配置 目前支持的集群容错策略有：

- **Failover 失效切换（缺省）**


``` xml
<motan:protocol ... haStrategy="failover"/>
```

>失败自动切换，当出现失败，重试其它服务器。

- **Failfast 快速失败**

``` xml
<motan:protocol ... haStrategy="failfast"/>
```

>快速失败，只发起一次调用，失败立即报错。

## dubbo的容错策略

motan实际上是dubbo的子集，也可以说是阉割版。一下是dubbo支持的容错策略。

- **1.failover cluster**

 >失败自动切换，当出现失败，重试其他服务器（缺省），通常用于读操作，但重试会带来更长的延时，可通过retries=“2”来设置重试次数（不含第一次）

``` xml
<dubbo:service retries="2">
```

- **2.failfast cluster**

>快速失效，只发起一次调用，失败立即报错。通常用于非幂等性写操作，比如说新增记录

``` xml
<dubbo:service cluster="failfast">
```

- **3.failsaft cluster**

 >失败安全，出现异常时，直接忽略，通常用于写入审计日志等操作

``` xml
<dubbo:service cluster="failsafe">
```

- **4.failback cluster**

> 失败自动恢复，后台记录失败请求，定时重发，通常用于消息通知操作

``` xml
<dubbo:service cluster="failback">
```

- **5.forking cluster**

 >并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多的服务器资源。可通过forks=“2”来设置最大并行数。

``` xml
<dubbo:service cluster="forking">
```

## motan容错策略源代码解读

motan的容错策略尽管支持的不够多，但是对于大部分场景已经够用。而对于一些特殊场景我们可以基于motan 的SPI机制扩展实现自己需要的容错策略。

motan的容错策略及时基于java的`策略模式`实现的。

>策略模式作为一种软件设计模式，指对象有某个行为，但是在不同的场景中，该行为有不同的实现算法。比如每个人都要“交个人所得税”，但是“在美国交个人所得税”和“在中国交个人所得税”就有不同的算税方法。

motan容错的接口：

``` java
// 可以通过motan的SPI机制扩展
@Spi(scope = Scope.PROTOTYPE)
public interface HaStrategy<T> {
    // 从后面的实现来看，没发现有什么用
    void setUrl(URL url);
    //基于具体的LoadBalance调用请求，问答模式，一个request返回一个response，
    Response call(Request request, LoadBalance<T> loadBalance);
}

```

AbstractHaStrategy这个抽象方法做的事情貌似没什么用，直接看具体的实现：
failfast很简单，直接略去。看failover的实现：

``` java

@SpiMeta(name = "failover")
public class FailoverHaStrategy<T> extends AbstractHaStrategy<T> {

    //确保每个线程持有一份单独的ArrayList<Referer<T>（FailoverHaStrategy，会有单个实例被多个线程调用）
    protected ThreadLocal<List<Referer<T>>> referersHolder = new ThreadLocal<List<Referer<T>>>() {
        @Override
        protected java.util.List<com.weibo.api.motan.rpc.Referer<T>> initialValue() {
            return new ArrayList<Referer<T>>();
        }
    };

    @Override
    public Response call(Request request, LoadBalance<T> loadBalance) {

        List<Referer<T>> referers = selectReferers(request, loadBalance);
        if (referers.isEmpty()) {
            throw new MotanServiceException(String.format("FailoverHaStrategy No referers for request:%s, loadbalance:%s", request,
                    loadBalance));
        }
        URL refUrl = referers.get(0).getUrl();
        // 先使用method的配置
        int tryCount =
                refUrl.getMethodParameter(request.getMethodName(), request.getParamtersDesc(), URLParamType.retries.getName(),
                        URLParamType.retries.getIntValue());
        // 如果有问题，则设置为不重试
        if (tryCount < 0) {
            tryCount = 0;
        }

        for (int i = 0; i <= tryCount; i++) {
            Referer<T> refer = referers.get(i % referers.size());
            try {
                request.setRetries(i);
                return refer.call(request);
            } catch (RuntimeException e) {
                // 对于业务异常，直接抛出
                if (ExceptionUtil.isBizException(e)) {
                    throw e;
                } else if (i >= tryCount) {
                    throw e;
                }
                LoggerUtil.warn(String.format("FailoverHaStrategy Call false for request:%s error=%s", request, e.getMessage()));
            }
        }
      // 从逻辑来看 代码不会到这个地方（如果是业务逻辑直接抛出，如果达到重试次数，也直接抛出异常了二中断了）
        throw new MotanFrameworkException("FailoverHaStrategy.call should not come here!");
    }

    /**
     * 先从threadLocal拿到当前线程持有的List<Referer<T>> referers，然后清除，再从loadbalance哪里复制一份最新的引用添加进去
     * haStratege能感知到引用列表的实时变化
     * @param request
     * @param loadBalance
     * @return
     */
    protected List<Referer<T>> selectReferers(Request request, LoadBalance<T> loadBalance) {
        List<Referer<T>> referers = referersHolder.get();
        referers.clear();
        loadBalance.selectToHolder(request, referers);
        return referers;
    }

}
```


这段代码花费了些时间理解。

- 1.为什么要将引用放到new ThreadLocal<List<Referer<T>>>()?

>首先单个HAStratege会被多个线程调用，加入ThreadLocal必然是为了保证每个线程持有单独的一份refererList。
>如果没有ThreadLocal，当其他线程调用selectReferers()会改变refererList，这个时候call方法中的循环会有问题，临界条件可能会抛异常。

- 2.为什么需要selectReferers？

>这个方法可以确保referer列表的实时性，能够实时感知到referer列表的变化。


## 关于容错的其他考虑
上文讨论的容错都是基于业务调用方考虑的一些策略。无论是motan还是dubbo都没有基于服务端自身保护的考虑。

>当企业微服务化以后，服务之间会有错综复杂的依赖关系，例如，一个前端请求一般会依赖于多个后端服务。在实际生产环境中，服务往往不是百分百可靠，服务可能会出错或者产生延迟，如果一个应用不能对其依赖的故障进行容错和隔离，那么该应用本身就处在被拖垮的风险中。在一个高流量的网站中，某个单一后端一旦发生延迟，可能在数秒内导致所有应用资源(线程，队列等)被耗尽，造成所谓的雪崩效应(Cascading Failure)，严重时可致整个网站瘫痪。

- **电路熔断器模式(Circuit Breaker Patten),** 该模式的原理类似于家里的电路熔断器，如果家里的电路发生短路，熔断器能够主动熔断电路，以避免灾难性损失。在分布式系统中应用电路熔断器模式后，当目标服务慢或者大量超时，调用方能够主动熔断，以防止服务被进一步拖垮；如果情况又好转了，电路又能自动恢复，这就是所谓的弹性容错，系统有自恢复能力。下图Fig 8是一个典型的具备弹性恢复能力的电路保护器状态图，正常状态下，电路处于关闭状态(Closed)，如果调用持续出错或者超时，电路被打开进入熔断状态(Open)，后续一段时间内的所有调用都会被拒绝(Fail Fast)，一段时间以后，保护器会尝试进入半熔断状态(Half-Open)，允许少量请求进来尝试，如果调用仍然失败，则回到熔断状态，如果调用成功，则回到电路闭合状态。

- **舱壁隔离模式(Bulkhead Isolation Pattern)**，顾名思义，该模式像舱壁一样对资源或失败单元进行隔离，如果一个船舱破了进水，只损失一个船舱，其它船舱可以不受影响 。线程隔离(Thread Isolation)就是舱壁隔离模式的一个例子，假定一个应用程序A调用了Svc1/Svc2/Svc3三个服务，且部署A的容器一共有120个工作线程，采用线程隔离机制，可以给对Svc1/Svc2/Svc3的调用各分配40个线程，当Svc2慢了，给Svc2分配的40个线程因慢而阻塞并最终耗尽，线程隔离可以保证给Svc1/Svc3分配的80个线程可以不受影响，如果没有这种隔离机制，当Svc2慢的时候，120个工作线程会很快全部被对Svc2的调用吃光，整个应用程序会全部慢下来。

- **限流(Rate Limiting/Load Shedder)**，服务总有容量限制，没有限流机制的服务很容易在突发流量(秒杀，双十一)时被冲垮。限流通常指对服务限定并发访问量，比如单位时间只允许100个并发调用，对超过这个限制的请求要拒绝并回退。

- **回退(fallback)**，在熔断或者限流发生的时候，应用程序的后续处理逻辑是什么？回退是系统的弹性恢复能力，常见的处理策略有，直接抛出异常，也称快速失败(Fail Fast)，也可以返回空值或缺省值，还可以返回备份数据，如果主服务熔断了，可以从备份服务获取数据。

>Netflix将上述容错模式和最佳实践集成到一个称为`Hystrix`的开源组件中，凡是需要容错的依赖点(服务，缓存，数据库访问等)，开发人员只需要将调用封装在Hystrix Command里头，则相关调用就自动置于Hystrix的弹性容错保护之下。Hystrix组件已经在Netflix经过多年运维验证，是Netflix微服务平台稳定性和弹性的基石，正逐渐被社区接受为标准容错组件。

## 小结

容灾策略motan实现的比较简单 ，代码也相对好懂，可以参考借鉴。

-----
## 后记：
希望有时间可以看Hystrix的实现，