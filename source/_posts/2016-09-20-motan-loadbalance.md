---
layout: post
title:  "Motan源码解读-负载均衡"
date:  2016-09-20
categories: [Motan源码解读]
keywords: motan,源码,loadbalance,负载均衡
---

## 负载均衡的策略

>Motan 在集群负载均衡时，提供了多种方案，缺省为 ActiveWeight，并支持自定义扩展。 负载均衡策略在Client端生效，因此需在Client端添加配置

**目前支持的负载均衡策略有：**

- ActiveWeight(缺省)

```xml
<motan:protocol ... loadbalance="activeWeight"/>
```

>低并发度优先： referer 的某时刻的 call 数越小优先级越高
由于 Referer List 可能很多，比如上百台，如果每次都要从这上百个 Referer 或者最低并发的几个，性能有些损耗，因此 random.nextInt(list.size()) 获取一个起始的 index，然后获取最多不超过 MAX_REFERER_COUNT 的状态是 isAvailable 的 referer 进行判断 activeCount.

- Random

```xml
<motan:protocol ... loadbalance="random"/>
```

>随机，按权重设置随机概率。
在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

- RoundRobin

```xml
<motan:protocol ... loadbalance="roundrobin"/>
```

>轮循，按公约后的权重设置轮循比率

- LocalFirst

```xml
<motan:protocol ... loadbalance="localFirst"/>
```

>本地服务优先获取策略，对referers根据ip顺序查找本地服务，多存在多个本地服务，获取Active最小的本地服务进行服务。
当不存在本地服务，但是存在远程RPC服务，则根据ActivWeight获取远程RPC服务
当两者都存在，所有本地服务都应优先于远程服务，本地RPC服务与远程RPC服务内部则根据ActiveWeight进行

- Consistent

```xml
<motan:protocol ... loadbalance="consistent"/>
```

>一致性 Hash，相同参数的请求总是发到同一提供者

- ConfigurableWeight

```xml
<motan:protocol ... loadbalance="configurableWeight"/>
```

>权重可配置的负载均衡策略

## motan加载loadbalance策略的机制

上面我们了解了motan支持的负载均衡的策略，下面我们来剖析一下motan怎么实现的。

RPCClient首先会经过一层HA（这里 motan实现了两种容灾策略，failfast&failover），然后会根据配置的loadbalance的策略选取相应的nettyClient发送请求到RPCServer。流程如下图所示：

![](/images/motan/14612385789967.jpg)



motan的loadbalance策略实现在客户端，客户端初始化时会根据配置，基于spi机制查找对应的实现（不了解SPI的可以看[Motan源码解读-SPI机制](http://zhizus.com/code/motan-spi)加载loadbalance的实现。
调用链如下图所示：

![](/images/motan/motan-lb.png)

## motan LoadBalance实现

![](/images/motan/loadbalance.png)

接下来，我们看loadbalance接口

```java
// 可以基于SPI扩展，使用方可以实现自己的LoadBalance
@Spi(scope = Scope.PROTOTYPE)
public interface LoadBalance<T> {

    //当可用的应用列表变化时会调用这个方法刷新（motan的可用引用列表是基于服务发现这种模式实现，目前实现了consul&zk）
    void onRefresh(List<Referer<T>> referers);

    // 基于负载均衡的策略选择可用的引用
    Referer<T> select(Request request);

    // FailoverHaStrategy会使用到这个，多线程场景
    void selectToHolder(Request request, List<Referer<T>> refersHolder);

    // ?? 仅仅属于WeightLoadBalance的特性，定义在最上层接口是否合适？？
    //cluster.getLoadBalance().setWeightString(weights); 这里使用的时候判断一下类型，对WeightLoadBalance这种类型单独处理向下转型是否可以？
    // 或者是否有其他更好的办法？？
    void setWeightString(String weightString);

}

```

下面我们AbstractLoadBalance,实现了公有的基础逻辑。将不同的策略交由子类去实现，实际上应用到了java设计模式的模板模式。

>模板模式在模板模式（Template Pattern）中，一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。这种类型的设计模式属于行为型模式。

下面我们看代码：

``` java
public abstract class AbstractLoadBalance<T> implements LoadBalance<T> {

    private List<Referer<T>> referers;

    @Override
    public void onRefresh(List<Referer<T>> referers) {
        // 只能引用替换，不能进行referers update。
        this.referers = referers;
    }

    @Override
    public Referer<T> select(Request request) {
        List<Referer<T>> referers = this.referers;

        Referer<T> ref = null;
        if (referers.size() > 1) {
            ref = doSelect(request);

        } else if (referers.size() == 1) {
            ref = referers.get(0).isAvailable() ? referers.get(0) : null;
        }

        if (ref != null) {
            return ref;
        }
        throw new MotanServiceException(this.getClass().getSimpleName() + " No available referers for call request:" + request);
    }

    @Override
    public void selectToHolder(Request request, List<Referer<T>> refersHolder) {
        List<Referer<T>> referers = this.referers;

        if (referers == null) {
            throw new MotanServiceException(this.getClass().getSimpleName() + " No available referers for call : referers_size= 0 "
                    + MotanFrameworkUtil.toString(request));
        }

        if (referers.size() > 1) {
            doSelectToHolder(request, refersHolder);

        } else if (referers.size() == 1 && referers.get(0).isAvailable()) {
            refersHolder.add(referers.get(0));
        }
        if (refersHolder.isEmpty()) {
            throw new MotanServiceException(this.getClass().getSimpleName() + " No available referers for call : referers_size="
                    + referers.size() + " " + MotanFrameworkUtil.toString(request));
        }
    }

    protected List<Referer<T>> getReferers() {
        return referers;
    }

    @Override
    public void setWeightString(String weightString) {
        LoggerUtil.info("ignore weightString:" + weightString);
    }

    protected abstract Referer<T> doSelect(Request request);

    protected abstract void doSelectToHolder(Request request, List<Referer<T>> refersHolder);
}


```

子类重写`doSelect(Request request)`方法就是实现自己的策略。(里面还有一个doSelectToHolder(Request request, List<Referer<T>> refersHolder)方法，这个FailoverHaStrategy会涉及到，我们下一篇在讨论)

看了一遍各种策略具体实现，貌似没什么好说的。有兴趣可以直接看motan源代码。

下面贴出一个另外一种的实现，仅供学习参考，`非线程安全`，使用时需要注意.

``` java
public class LoadBalance {
    private int i = 0;
    private int cw = 0;
    private int[] weight;
    private int count;

    public LoadBalance(int count) {
        this.count = count;
    }

    public LoadBalance(int[] weight) {
        this.count = weight.length;
        this.weight = weight;
    }

    public int hashIndex(String key) {
        long hash = 5381L;

        int index;
        for(index = 0; index < key.length(); ++index) {
            hash = (hash << 5) + hash + (long)key.charAt(index);
            hash &= 4294967295L;
        }
        index = (int)hash % this.count;
        index = Math.abs(index);
        return index;
    }
    public int roundIndexByWeight() {
        do {
            this.i = (this.i + 1) % this.count;
            if(this.i == 0) {
                this.cw -= this.gcd();
                if(this.cw <= 0) {
                    this.cw = this.max();
                    if(this.cw == 0) {
                        return 0;
                    }
                }
            }
        } while(this.weight[this.i] < this.cw);
        return this.i;
    }
    public int roundIndex() {
        int j = this.i;
        j = (j + 1) % this.count;
        this.i = j;
        return this.i;
    }
    private int gcd() {
        BigInteger value = null;
        if(this.weight.length > 0) {
            value = BigInteger.valueOf((long)this.weight[this.i]);
        }
        for(int i = 0; i < this.weight.length - 1; ++i) {
            BigInteger tmp = BigInteger.valueOf((long)this.weight[i]);
            tmp = tmp.gcd(BigInteger.valueOf((long)this.weight[i + 1]));
            if(value.compareTo(tmp) > 0) {
                value = tmp;
            }
        }
        if(null != value) {
            return value.intValue();
        } else {
            return 0;
        }
    }
    private int max() {
        int value = 0;
        if(this.weight.length > 0) {
            value = this.weight[0];
        }

        for(int i = 0; i < this.weight.length - 1; ++i) {
            int tmp = this.weight[i];
            if(value < tmp) {
                value = tmp;
            }
        }
        return value;
    }
}
```

``` java
public abstract class HALBProxy {
    private HALBProxy.Strategy strategy;
    private LoadBalance lb;
    public HALBProxy(HALBProxy.Strategy strategy) {
        this.logger = Logger.getLogger(this.getClass());
        this.availServers = new CopyOnWriteArrayList();
        this.unavailServers = new CopyOnWriteArrayList();
        this.listeners = new HashSet();
        this.strategy = strategy;
    }
    public int getIndex() {
        switch(strategy) {
        case 1:
            return 0;
        case 2:
            return this.lb.roundIndex();
        case 3:
            return this.lb.roundIndexByWeight();
        default:
            return 0;
        }
    }
    public int getIndex(String key) {
        return this.lb.hashIndex(key);
    }
     public static enum Strategy {
        DEFAULT,
        RR,
        WRR,
        HASH;
        private Strategy() {
        }
    }
}


```

## 小结
总的来说负载均衡部分还算比较简单，LocalFirst优先读取本机的服务这个策略还是很不错的。

---

关于更多loadbalance请看[老外的文章(直戳loadbalance本质)](http://tutorials.jenkov.com/software-architecture/load-balancing.html)

